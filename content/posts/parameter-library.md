---
title: "Serializing and Deserializing external data in Elixir"
date: 2022-12-30T10:11:21+02:00
draft: true
---


Since I started using Elixir I always felt that the ecosystem was missing a good library for serializing and deserializing external data. We already have some libraries in this area but it always feels that the existing ones are missing some features. For instance, when serializing parameters a library should have the following features:
- Schema definition to shape your data.
- Type definition to convert the external data field to internal types.
- Common validation methods.
- Function for serializing and deserializing the data by the given schema.
- Flexible and extensible.


In this article, I will dive into how commonly this is done in most of the Elixir apps and how we can improve that.

## External data
External data means any type of data that comes into your application and you don't completely know about it. This data can be unpredictable, maybe coming with unknown values or even wrong types. This can potentially harm your system if you don't parse it correctly. One of the common examples of dealing with external data is when building an API. The controller us usually built like this:

```elixir
defmodule MyAppWeb.SuperController do
  use MyAppWeb, :controller

  def index(conn, _params) do
    supers = Context.list_users()
    render(conn, :index, users: users)
  end

  def create(conn, %{"user" => user_params}) do
    with {:ok, %User{} = user} <- Context.create_user(user_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", ~p"/api/users/#{user}")
      |> render(:show, user: user)
    end
  end
end
```

From the controller create action, `user_params` comes from external data.

This approach looks fine for most cases but there are two small problems:
- We infer that the database schema will be almost the same as the API request/response.
- We are not validating the data that are coming into our system on the **create** function in the controller.

We may rely on that for validating data we should use the changesets from `Ecto` and this is a correct statement, we should validate the data before saving it on the database. But what if our API changes over time but the database schema doesn't? 
These validations will start to get trickier and using the same DB schema with Changesets starts to become more troublesome to solve complex cases when dealing with external parameters.

To outcome this problem it's possible to use [Embedded Schemas](https://hexdocs.pm/ecto/embedded-schemas.html) and create a different schema for your API. This will solve most of the problems described above.
The reason this is enough is that we control and shape our API schema, this is internal to our application so we can define its boundaries. Now, what would happen if we would like to apply the same pattern when integrating into an external API?

## Integrating with External API

Let's say that your Elixir application needs to integrate with an external payment provider. This payment provider is not famous and while testing its endpoints, we noticed some inconsistencies in its API docs. This situation is very common when integrating with third-party softwares.
For this fictional API, this is a POST request example for charging the user:

```bash
# POST /charge-user
# Request
curl -v -X POST https://api-payment-fictional.com/charge-user \
-H "Content-Type: application/json" \
-d '{
  "detail": {
    "invoiceNumber": "ABC123",
    "currencyCode": "USD",
    "amount": "500",
    "discount": "10%",
    "user": {
      "firstName": "John",
      "lastName": "Doe",
      "phones": [
        {
          "countryCode": "001",
          "number": "123456789",
          "phoneType": "PERSONAL"
        }
      ]
    }
  }
}'

# Response
'{
  "id": "1234",
  "status": "NEW",
  "detail": {
    "amount": "500.00",
    "billed": "450.00",
    "currencyCode": "USD"
}'
```

This would be my approaches for dealing with this data:
- Create a client module to have the request/response implementation of this payment provider endpoint.
- Create helper functions to shape this data into elixir internal types and do proper validation for the fields.

How do we properly shape and validate this data? By using schema definitions! Again we can use [Embedded Schemas](https://hexdocs.pm/ecto/embedded-schemas.html) for that, let's give a try:

```elixir
# Request
defmodule MyApp.ChargeRequest do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    embeds_one :details, Detail do
      field :invoiceNumber, :boolean
      field :currencyCode, :string
      field :amount, :decimal
      field :discount, :string
      embeds_one :user, User do
        field :firstName, :string
        field :lastName, :string
        embeds_many :phones, Phone do
          field :countryCode, :string
          field :number, :string
          field :phoneType, Ecto.Enum, values: ["PERSONAL", "WORK"]
        end
      end
    end
  end

  def load(params) do
    params
    |> changeset()
    |> apply_action(:update)
  end

  def changeset(params) do
    %__MODULE__{}
    |> cast_embed(:detail, required: true, with: &detail_changeset/2)
  end

  defp detail_changeset(schema, params) do
    schema
    |> cast(params, [:invoiceNumber, :currencyCode, :amount, :discount])
    |> validate_required([:invoiceNumber, :currencyCode, :amount, :discount])
    |> cast_embed(:user, required: true, with: &user_changeset/2)
  end

  defp user_changeset(schema, params) do
    schema
    |> cast(params, [:firstName, :lastName])
    |> validate_required([:firstName, :lastName])
    |> cast_embed(:phones, required: true, with: &phone_changeset/2)
  end

  defp phone_changeset(schema, params) do
    schema
    |> cast(params, [:countryCode, :number])
    |> validate_required([:countryCode, :number])
  end
end
```

```elixir
# Response
defmodule MyApp.ChargeResponse do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :id, :string
    field :status, Ecto.Enum, values: ["NEW", "CHARGED", "CANCELED"]
    embeds_one :detail, Detail do
      field :amount, :decimal
      field :currencyCode, :string
    end
  end

  def load(params) do
    params
    |> changeset()
    |> apply_action(:update)
  end

  def changeset(params) do
    %__MODULE__{}
    |> cast(params, [:id, :status])
    |> validate_required([:id, :status, :amount, :discount])
    |> cast_embed(:detail, required: true, with: &detail_changeset/2)
  end

  defp detail_changeset(schema, params) do
    schema
    |> cast(params, [:amount, :currencyCode])
    |> validate_required([:amount, :currencyCode])
  end
end
```

Now as an example, the client module using (Tesla)[https://github.com/elixir-tesla/tesla] as HTTP client:

```elixir
defmodule MyApp.PaymentAPI do
  use Tesla

  alias MyApp.ChargeRequest
  alias MyApp.ChargeResponse

  plug Tesla.Middleware.BaseUrl, "https://api-payment-fictional.com/"
  plug Tesla.Middleware.JSON

  def charge_user(params) do
    with {:ok, data} <- ChargeRequest.load(params) do
      post("/charge-user", data)
      |> parse_charge_user_response()
    end
  end

  defp parse_charge_user_response({:ok, response}) do
    ChargeResponse.load(response.body)
  end

  defp parse_charge_user_response({:error, _error} = error) do
    error
  end
end
```

And now the API module is ready to be used in your other modules for requesting the charge user endpoint and successfully parsing both the request and responses. It's always important to parse the data that comes from external APIs because this data will be used in the application logic. For this case, parsing the data incorrectly could mean that you are not charging the user with the correct amount or you are not successfully validating if a user was already charged or not.

Now let's show some cons of this current implementation:
- The schemas are very verbose, if you implement more endpoints you will have more verbose schemas.
- Schemas are loaded into structs, this is not ideal in APIs where you can omit some fields when doing the request. Using structs the omitted fields on params will be evaluated as `nil`.
- To follow the APIs JSON keys, the schemas have camelCase fields which under the hood will create a struct with the same case. This is not common to do in Elixir.

Next, let's see how we can improve this using the [Parameter](https://github.com/phcurado/parameter) library specially designed for this problem.
