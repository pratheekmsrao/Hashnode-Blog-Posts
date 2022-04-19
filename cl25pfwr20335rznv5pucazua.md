## Test Your FastAPI API with Pytest

Writing test cases for your code is a tedious task and most developers ignore this part.
But writing good test cases for your program saves a ton of effort in testing your application.
If you have a CI/CD pipeline for your application, automated tests are an integral part of your pipeline.

# Introduction

In my previous [blog](https://pratheekms.hashnode.dev/crud-api-using-python-fastapi-and-mongodb), I explained how to create a simple CRUD API using FastAPI and MongoDB.

In this guide let's write test cases for that.

In python, many frameworks for testing like UnitTest, Robot, Pytest, etc. We are using the most popular one, [pytest](https://docs.pytest.org/). 

By using pytest we can write more readable and scalable tests.

Before testing out API, let's try to understand how pytest can be used to write tests.
## Pytest example
we have a function to add two numbers:

```
#add.py
def add_numbers(num1,num2):
    return num1+num2
``` 
This is a go-to scenario for all the examples, let's test this.

```
#test_add.py
import pytest
from add import add_numbers
def test_add_numbers():
    total=add_numbers(2,3)
    assert total==5
``` 
For pytest to recognize the tests, all the test files should start with `test_` and all the test functions should start with `test_`

In the above example, the file name is `test_add.py` and the test function name is `test_add_numbers`.

first, we import the module pytest and define a test function. then out add function `add_numbers`.
our test function calls `add_number` with arguments `2,3`.

The sum is returned and stored in `total`. Now we use `assert` to test whether we get the right value or not. so we assert a total of 5. 

To run the test, we use pytest command followed by the file name.
`pytest test_add.py` if everything goes right, you'll see something like below.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1649933571539/qOFkW7K9K.png)

In the above image, each green dot represents one test case. You probably ought two green dots but we have only one test case. No worries let's write one more.

What happens if I pass a string value and a number to our add function, it probably throughs `TypeError`. 
We capture that in our test case like the below example.

```
def test_add_numbers_with_invalid_data():
    with pytest.raises(TypeError):
        add_numbers('2',3)
``` 
We call the function within `pytest.raises` and pass the exception i.e `TypeError` to it.

## Testing the API with pytest

Our CRUD [API](https://github.com/pratheekmsrao/fastapi-mongo-CRUD) is built using the FaspApi, and FastApi comes with its own test client which is built on pytest.

We create a file `test_app.py` to test the app.

```
#test_app.py

#import the FastApi test client
from fastapi.testclient import TestClient

#import app from main
from main import app

#instanciate the test client
client = TestClient(app)

def test_home():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json()["message"] == "Welcome Home!!"
``` 

Behind the scene, `TestClient` creates a test server and runs the `app`. Then client makes the `get` call to API and stores the response.
Then we assert the response's status_code  and message.

## Mocking the MongoDB

We are using MongoDB as a database to store our Employee data.
While testing we can use the test database for all our tests. But in this case, we will learn a new concept of mocking.

Mocking is the way of replicating a functionality. By using mocking we control the functionality and be free from unexpected errors.

We use [mongomock](https://github.com/mongomock/mongomock) to mock the MongoDB.

It is best practice to write the mock code in `conftest.py`, so pytest runs this file before running the tests. So that our mocked MongoDB will be available to testing.


```
#conftest.py

import pytest
import mongomock

from main import Employee

@pytest.fixture()
def mongo_mock(monkeypatch):
    client = mongomock.MongoClient()
    db = client.get_database("EmployeeDB")
    col = db.get_collection("Employee")
    emp_data: Employee = {
        "id": 1,
        "name": "test_user",
        "email": "test_user@gmail.com",
        "profession": "tester",
        "level": "A1",
    }

    col.insert_one(emp_data)

    def fake_db():
        return db

    monkeypatch.setattr("main.get_db", fake_db)
``` 
`@pytest.fixture()` is a fixture, that can be used in tests. 
we use `monkeypatch`, this will patch our `get_db` function with `fake_db`.
This seems confusing, I'll it better with the test.

## Testing the endpoints
### GET get_employee
Below is the test to get employee endpoint.

```
def test_get_employee(mongo_mock):
    response = client.get("/employee/1")
    assert response.status_code == 200
    assert response.json()["id"] == 1
    assert response.json()["name"] == "test_user"
    assert response.json()["email"] == "test_user@gmail.com"
    assert response.json()["profession"] == "tester"
    assert response.json()["level"] == "A1"
``` 
We can see our fixture `mongo_mock` is passed as a parameter.

when we run the `pytest` command, pytest searches for the `conftest.py` file and creates all the fixtures.
Our contest has `mongo_mock` as a fixture. This creates an instance of `mongomock.MongoClient()`, creates a DB `EmployeeDB` and a collection `Employee`. It also inserts one record into the collection.
So we have our mocked MongoDB with a fake record. 

But our `get_db` function connects to the real MongoDB and returns the `db` object.
We don't need the real DB client, we need our mocked DB client. 

Now, `monkeypatch` comes into the picture. whenever our API code executes the `get_db` function we replace the return object with our mocked DB object. So our tests don't use the real database instead use mocked database.

Our test function `test_get_employee` calls the API and tries to get the employee details for id=1.
we already have an employee with id=1 in our mocked MongoDB and return the value.

### POST create_employee

we test creating a new employee with a dummy data


```
emp_data = {
    "id": 123,
    "name": "test_user",
    "email": "test_user@gmail.com",
    "profession": "Tester",
    "level": "A1",
}

def test_create_employee(mongo_mock):
    response = client.post("/employee", data=json.dumps(emp_data))
    assert response.status_code == 200
    assert response.json()["message"] == f"Employee {emp_data.get('name')} created"
``` 
### PUT update_employee

we test the updated employee by creating an updated record data and passing it while the API call.

Remember we are updating the employee with id=1, because that is the only record that is available in our mocked database.
```
updated_emp_data = {"id": 1, "profession": "Senior Tester", "level": "A5"}

def test_update_employee(mongo_mock):
    response = client.put("/employee", data=json.dumps(updated_emp_data))
    assert response.status_code == 200
    assert response.json()["message"] == "Employee updated."
``` 

### DELETE delete_employee

Let's delete the employee created by the fixture.

```
def test_delete_employee(mongo_mock):
    response = client.delete("employee/1")
    assert response.status_code == 200
    assert response.json()["message"] == "Employee deleted"
``` 


