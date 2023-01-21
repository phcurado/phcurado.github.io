---
title: "Using Parameter for dealing with API request and responses"
date: 2023-01-21T17:53:41+02:00
draft: false
---

This is a continuation from the previous article [Serializing and Deserializing external data in Elixir]({{< ref "serializing-and-deserializing-external-data-in-elixir" >}}).

As previously shown, it's verbose and not always ideal using Ecto Embedded Schemas for loading and validating external parameters. I recently created a library that help solving the problems described on the previous article. Let's reimplement the API schemas using [Parameter](https://github.com/phcurado/parameter) and see what benefits it will bring.

## Modeling the data from an API

Considering the below example on how an API endpoint documentation may look like:

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
      "lastName": "Doe"
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

We can shape this data in two schemas (request and response) using the [Parameter](https://github.com/phcurado/parameter) library:


```elixir
# Request
defmodule MyApp.ChargeRequest do
  use Parameter.Schema
  # Parameter by default makes the fields optional but setting this flag
  # will make this behaviour change for all fields
  @fields_required true

  param do
    has_one :details, Detail do
      field :invoice_number, :string, key: "invoiceNumber"
      field :currency_code, :string, key: "currencyCode"
      field :amount, :decimal
      field :discount, :string
      has_one :user, User do
        field :first_name, :string, key: "firstName"
        field :last_name, :string, key: "lastName"
      end
    end
  end
end
```

```elixir
# Response
defmodule MyApp.ChargeResponse do
  use Parameter.Schema

  @fields_required true

  enum Status do
    value :new, key: "NEW"
    value :charged, key: "CHARGED"
    value :canceled, key: "CANCELED"
  end

  param do
    field :id, :string
    field :status, MyApp.ChargeResponse.Status
    has_one :detail, Detail do
      field :amount, :decimal
      field :currency_code, :string, key: "currencyCode"
    end
  end
end
```

We mapped the API request/response by defining the fields in a very similar way on how we do for Ecto Schemas. The main difference is that we declared the exact field key of the JSON docs in the field definition without using camelCase in elixir code. Also adding the `@fields_required true` flag in the module will make all fields required and they will be properly evaluated when loading the parameters. 
Now let's create the HTTP client module for making the request to the external API:

```elixir
defmodule MyApp.PaymentAPI do
  use Tesla

  alias MyApp.ChargeRequest
  alias MyApp.ChargeResponse

  plug Tesla.Middleware.BaseUrl, "https://api-payment-fictional.com/"
  plug Tesla.Middleware.JSON

  def charge_user(params) do
    with :ok <- Parameter.validate(ChargeRequest, params),
         {:ok, data} <- Parameter.dump(ChargeRequest, params) do
      post("/charge-user", data)
      |> parse_charge_user_response()
    end
  end

  defp parse_charge_user_response({:ok, response}) do
    Parameter.load(ChargeResponse, response.body)
  end

  defp parse_charge_user_response({:error, _error} = error) do
    error
  end
end
```

Now with Parameter, we are successfully validating request and response parameters, these are the main benefits:
- We can dump (deserialize) or load (serialize) the data using internal elixir types and correctly converts the given key/value of external data schema.
- No need to add changeset functions for each schema and association. Parameter can infer everything by just having the schema definition and the parameters to parse.
- Parameter can load the schemas as Maps or Struct depending on how you want the data internally. This is easily done by adding **struct: true** in the third argument of the **load** function (ex: **Parameter.load(ChargeResponse, response.body, struct: true)**

Now let's test this implementation and see the results.

First, we will simulate the params coming in the charge functions and validate them:
```elixir
iex> params = %{
  details: %{
    amount: Decimal.new("500"),
    currency_code: "USD",
    discount: "10%",
    invoice_number: "ABC123",
    user: %{
      first_name: "John",
      last_name: "Doe"
    }
  }
}

iex> Parameter.validate(MyApp.ChargeRequest, params)
:ok
```

Now with the validated parameter, we will dump its value to the API schema
```elixir
iex> Parameter.dump(MyApp.ChargeRequest, params)
{:ok,
 %{
   "details" => %{
     "amount" => #Decimal<500>,
     "currencyCode" => "USD",
     "discount" => "10%",
     "invoiceNumber" => "ABC123",
     "user" => %{
       "firstName" => "John",
       "lastName" => "Doe"
     }
   }
 }}
```

We can notice that all keys were converted to camelCase as per the schema definition. This means that we have the correct schema that the provider API requires.

For parsing the response, we can use the load function to load the external parameter following our schema definition:

```elixir
iex> params = %{
  "id" => "1234",
  "status" => "NEW",
  "detail" => %{
    "amount" => "500.00",
    "billed" => "450.00",
    "currencyCode" => "USD"
  }
}
iex> Parameter.load(MyApp.ChargeResponse, params)
{:ok,
 %{
   detail: %{currency_code: "USD", amount: #Decimal<500.00>},
   id: "1234",
   status: :new
 }}
```

We can see that again the camelCase from the API was correctly converted to the map keys defined on the ChargeResponse schema. Also, the values are correctly parsed to the given types and the status Enum value is converted to the corresponding atom value.

With this, we have predictable data with the correct type and more confidence to handle it on our application.

For all the possibilities that Parameter has to offer, check out the [official docs](https://hexdocs.pm/parameter/Parameter.html).

