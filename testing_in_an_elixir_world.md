slidenumbers: true

# **Testing** in an Elixir world
##### rafaelrochasilva@gmail.com
##### @RocRafael

---

# **Rafael Rocha**
- Full Stack developer at Plataformatec / The RealReal
- Master degree in Electrical Engineering
- Bachelor of Computer Science
- Father & Cooker

![right](rafael_rocha.jpg)

---
# Agenda

- Reflections on specifications and software development
- Base test concepts to create health software
- Testing in an Elixir world

---

# Reflections on **specifications** and software development

---

## When we start a **user story**, we read the description, the acceptance criteria, and start coding, but….

---

## How tests can reflect **specifications** and help us to build **confident code**?

---

## Are we bringing the specifications into code?

---

### Before **answering** those questions, we need some base testing **concepts** to create **health software**

---

# Why testing?

- Be **self-confident** about your deliverables

- Keep the project **costs low**

- Guarantee internal and external **quality**

---

# What are the types of tests?
**Acceptance:**
- Tests a functional requirement, usually by UI

**Integration:**
- Test between acceptance and unit, testing behavior of 2 or more entities together

**Unit:**
- Tests the behavior of 1 entity

---

# Test pyramid

![center](pyramid.jpg)

---

# What about Test Quality?

- Is the test verifying correct code behavior?
- Is it easy to understand?

---

# Firstly, let's get an example:

As a User, I want to fetch products from abcprincing.com so that we can store the current name and price of a given product in memory.

Acceptance Criteria:
- All products name and price should be fetched and stored in memory.
- The product name should be capitalized

---

### Let's use an Outside-in approach to guide us during the development.

Basically what we have to do are 3 tasks:

1) Fetch Products from the API
2) Build a tuple with id, capitalize name and price
3) Consume the data in memory

---

### Since we are doing an outside-in approach, the first thing we should do is create an Entity to fetch data from an external API.


### To keep the data in memory we will use Genserver.

---

# What is a Genserver?


“A GenServer is a process like any other Elixir process and it can be used to keep state, execute code asynchronously and so on."

---

```elixir
defmodule GreenBox.PriceUpdater do
  use GenServer

  @time_to_consume 10000 # 10 seconds

  def start_link do
    GenServer.start_link(__MODULE__, [])
  end

  @doc """
  Starts the Genserver to process the work, and schedule the next time the
  work will be done again
  """
  def init(_) do
    state = build_products()
    schedule_work()
    {:ok, state}
  end

  def list_products(pid) do
    GenServer.call(pid, :list_products)
  end
```

---

```elixir
  @doc """
  Run the job and reschedule it to run again after some time.
  """
  def handle_info(:get_products, _state) do
    products = build_products()

    schedule_work()
    {:noreply, products}
  end

  def handle_call(:list_products, _, state) do
    {:reply, state, state}
  end

  defp schedule_work() do
    Process.send_after(self(), :get_products, @time_to_consume)
  end
```

---

```elixir
 defp build_products() do
    fetch_products()
    |> proccess_products
  end

  defp proccess_products(products) do
    Enum.map(products, fn %{id: id, name: name, price: price} ->
      new_name = name |> String.downcase() |> String.capitalize()
      new_price = "$#{price/100}"
      %{
        id: id,
        name: new_name,
        price: new_price
      }
    end)
  end

  defp fetch_products do
    [
      %{id: "1234", name: "BLUE OCEAN CREAM", price: Enum.random(8000..10000)},
      %{id: "1235", name: "SEA SOAP", price: Enum.random(5000..60000)}
    ]
  end
end
```

---

# How can I test GenServer or Behaviours in General?

---

# Don't Test Elixir/OTP layers!!!

---

## What should I do?
## Should I avoid creating tests?

---

# Change your Design!

---

![](design.jpg)

---

## That been said, lets refactory our code and let guide us

---

# But before start doing the test, I want to share some REALLY valuable concepts about Test.

---

# Clarity

A test is **easy** to understand when we can see the **cause and consequence** between the **phases** of the test.

---

![](4phases.jpg)

---

# Let's see how it will work for our example

---

![](design.jpg)

---

# Extract GenServer code to a new entity

```elixir
 defp build_products() do
    fetch_products()
    |> proccess_products
  end

  defp proccess_products(products) do
    Enum.map(products, fn %{id: id, name: name, price: price} ->
      new_name = name |> String.downcase() |> String.capitalize()
      new_price = "$#{price/100}"
      %{
        id: id,
        name: new_name,
        price: new_price
      }
    end)
  end

  defp fetch_products do
    [
      %{id: "1234", name: "BLUE OCEAN CREAM", price: Enum.random(8000..10000)},
      %{id: "1235", name: "SEA SOAP", price: Enum.random(5000..60000)}
    ]
  end
end
```

---

# Let tests guide the development

```elixir
defmodule Greenbox.ProductFetcherTest do
  use ExUnit.Case, async: true
  alias Greenbox.ProductFetcher

  describe "Given a request to fetch a list of products" do
    test "builds a list of products with id, capitalized name and price in dollar" do
      products = ProductFetcher.build()
      assert [
        %{id: "1234", name: "Blue ocean cream", price: _},
        %{id: "1235", name: "Sea soap", price: _}
      ] = products
    end

    test "builds a product with the price with dollar sign" do
      product =
        ProductFetcher.build()
        |> List.first()

      assert Regex.match?(~r/\$\d+\.\d+/, product.price)
    end
  end
end
```

---

# Product Fetcher

```elixir
defmodule Greenbox.ProductFetcher do
  def build() do
    fetch_products()
    |> proccess_products()
  end

  defp fetch_products do
    [
      %{id: "1234", name: "BLUE OCEAN CREAM", price: Enum.random(8000..10000)},
      %{id: "1235", name: "SEA SOAP", price: Enum.random(5000..60000)}
    ]
  end

  defp proccess_products(products) do
    Enum.map(products, fn %{id: id, name: name, price: price} ->
      %{
        id: id,
        name: capitalize_name(name),
        price: price_to_money(price)
      }
    end)
  end
```

---

# Listen to your code: What about those 2 new functions?

```elixir
  defp price_to_money(price) do
    "$#{price / 100}"
  end

  defp capitalize_name(name) do
    name
    |> String.downcase()
    |> String.capitalize()
  end
end
```

---

# Build an entity to handle product structure
```elixir
defmodule Greenbox.ProductTest do
  use ExUnit.Case, async: true
  alias Greenbox.Product

  describe "Given a product" do
    test "transforms its name by capitalizing it" do
      # Setup
      product_name = "BLUE SOAP"

      # Exercise
      expected_product_name = Product.capitalize_name(product_name)

      # Verify
      assert expected_product_name == "Blue soap"
    end

    test "transforms the price in cents to dollar" do
      # Setup
      product_price_in_cents = 1253

      # Exercise
      expected_product_name = Product.price_to_money(product_price_in_cents)

      # Verify
      assert expected_product_name == "$12.53"
    end
  end
end
```

---

# Product Entity
```elixir
defmodule Greenbox.Product do
  defstruct [:id, :name, :price]

  def price_to_money(price) do
    "$#{price / 100}"
  end

  def capitalize_name(name) do
    name
    |> String.downcase()
    |> String.capitalize()
  end
end
```

---

![](design.jpg)

---

# Did you notice that we are hitting the Api every time we run our tests?


# What should we do?

---

# Since we don't want to hit the api while testing we need to figure out another way to test the ProductFetcher.

---

# Test Double, how to mock in Elixir?

---

# SUT and collaborator(DOC)
SUT: System Under Test
DOC: Collaborator

![](sut_doc.jpg)

---

# Test Double
Is the object that substitutes the real DOC
![](test_double.jpg)

---

Let's start by creating a fake client

---

# Fake client

```elixir
# test/support/fake_client.ex
defmodule Greenbox.FakeClient do

  def fetch_products do
    [
      %{id: "1234", name: "BLUE OCEAN CREAM", price: Enum.random(8000..10000)},
      %{id: "1235", name: "SEA SOAP", price: Enum.random(5000..60000)}
    ]
  end
end
```

---

# Configure the Fake Client
[.code-highlight: 3, 9-11]

```elixir
# lib/product_fetcher.ex
def build() do
  client = get_client()

  client.fetch_products()
  |> proccess_products()
end

defp get_client do
  Application.get_env(:greenbox, :abc_products_client)
end
```

---
# Configure the Fake Client
[.code-highlight: 9, 15-17]

```elixir
defmodule Greenbox.MixProject do
  use Mix.Project

  def project do
    [
      app: :greenbox,
      version: "0.1.0",
      elixir: "~> 1.7",
      elixirc_paths: elixirc_paths(Mix.env()),
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  # Specifies which paths to compile per environment.
  defp elixirc_paths(:test), do: ["lib", "test/support"]
  defp elixirc_paths(_), do: ["lib"]
end
```

---

```elixir
# config/test.exs
use Mix.Config

config :greenbox,
  abc_products_client: Greenbox.FakeClient
```

```elixir
# config/config.exs
config :greenbox,
  abc_products_client: Greenbox.ProductClient
```

---

# The Real Model that will do the call to the external API

```elixir
defmodule Greenbox.ProductClient do

  def fetch_products do
    # HTTPoison.get!("http://abcproducts.com")
  end
end
```
---

# OR Stub requests with libraries
- Bypass (https://github.com/PSPDFKit-labs/bypass)
- Mox (https://github.com/plataformatec/mox)


---

# What about Doctest?
# Are they supposed to substitute tests?

---
[.code-highlight: 4-9]
```elixir
defmodule Greenbox.Product do
  defstruct [:id, :name, :price]

  @doc """
  Converts price in cents to a string money format.
  ## Example:
    iex> Greenbox.Product.price_to_money(1245)
    "$12.45"
  """
  def price_to_money(price) do
    "$#{price / 100}"
  end
```

---

How tests can reflect **specifications**
and help us to build
**confident code**?

---

# Conclusion

- Always start outside-in
- Think in each scenario will need to reproduce your specification (Test Pyramid)
- Use mocks or build fake clients
- Don't test behaviors
- Abstract your code into Modules and made Unit Tests instead of testing behaviors

---

# Thank you!

https://github.com/rafaelrochasilva/greenbox

---

References:
