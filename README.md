# Mapping Object Relationships

## Learning Goals

- Implement ORM methods to manage a one-to-many relationship between classes.

---

## Key Vocab

- **Object-Relational Mapping (ORM)**: a programming technique that provides a
  mapping between an object-oriented data model and a relational database model.

---

## Introduction

In this lesson, we'll implement a one-to-many relationship between departments
and employees:

- An employee works in one department.
- A department has many employees.

![company erd](https://curriculum-content.s3.amazonaws.com/7134/python-p3-v2-orm/department_employee_erd.png)

Let's assume each employee works for exactly one department. The relationship
represents composition since an employee must be associated with a department.

## Code Along

This lesson is a code-along, so fork and clone the repo.

**NOTE: Remember to run `pipenv install` to install the dependencies and
`pipenv shell` to enter your virtual environment before running your code.**

```bash
pipenv install
pipenv shell
```

The `lib` folder contains starter code for the `Department` and `Employee`
classes. Both classes include ORM methods for mapping the individual class to a
corresponding table in the database:

| Python Class | Relational Database Table |
| ------------ | ------------------------- |
| Department   | departments               |
| Employee     | employees                 |

The one-to-many relationship between `Department` and `Employee` is **not**
implemented in the starter code. In this lesson, we will:

- Update the `Employee` class to add a new attribute to model the one-to-many
  relationship.
- Update the `Employee` class methods to store and manage the relationship in
  the "employees" database.
- Update the `Department` class to add a new method to **compute** (not store) a
  list of associated `Employee` class instances.

---

## Mapping a one-to-many relationship

The concept of **single source of truth** states we should design our software
and data to avoid redundancy. In terms of relational database design, we want to
store the relationship between two entities in just one place within the
database. We do this to avoid inconsistencies that often arise when redundant
data is not updated in a consistent manner.

When we have a relationship between two classes, we need to figure out where to
store the relationship. For the one-to-many relationship, the class (or entity)
on the "many" side is the owner of the relationship and is responsible for
storing the relationship. Why? Beside each object on the "many" side (i.e.
Employee) is related to just one entity on the "one" side (i.e. Department),
thus it only needs to store a single value to maintain the relationship.

![owning side](https://curriculum-content.s3.amazonaws.com/7134/python-p3-v2-orm/owning_side.png)

`Employee` is on the many side of the relationship, thus it **owns** the
relationship and is responsible for managing the relationship by storing a
single piece of data about the associated `Department`.

- The "Employee" class will have an attribute to reference the associated
  `Department` object.
- The "employees" table will contain a column to store a foreign key reference
  to the "departments" table.

`Department` is on the "one" side of the relationship, thus it should not
**store** a list of the associated `Employee` instances. Instead, a list of the
many `Employee` objects associated with the department will be **computed** by
looking at the `department_id` foreign key value stored with each employee.

---

## Updating the `Employee` class to own and manage the relationship

First let's update the `Employee` class to add a reference to the associated
`Department`.

- Import the `Department` class
- Update the `__init__` method to add a parameter `department` and store the
  value in a new attribute with the same name.
- Update the `__repr__` method to include the department name.

```py
from config import CURSOR, CONN
from department import Department

class Employee:

    def __init__(self, name, job_title, department, id=None):
        self.id = id
        self.name = name
        self.job_title = job_title
        self.department = department

    def __repr__(self):
        return (
            f"<Employee {self.id}: {self.name}, {self.job_title}, "
            + f"Department: {self.department.name} >")
```

Next, we'll update the `create_table` method to add a column named
`department_id` to store the relationship as a foreign key in the "employees"
table:

```py
@classmethod
def create_table(cls):
    sql = """
        CREATE TABLE IF NOT EXISTS employees (
        id INTEGER PRIMARY KEY,
        name TEXT,
        job_title TEXT,
        department_id INTEGER,
        FOREIGN KEY (department_id) REFERENCES departments(id))
    """
    CURSOR.execute(sql)
    CONN.commit()
```

We need up adapt the `save`, `update`, and `create` methods to persist the new
relationship by storing the department object's id (`self.department.id`) as the
foreign key value in the "employees" table:

```py
def save(self):
    sql = """
        INSERT INTO employees (name, job_title, department_id)
        VALUES (?, ?, ?)
    """

    CURSOR.execute(sql, (self.name, self.job_title, self.department.id))
    CONN.commit()

    self.id = CURSOR.lastrowid

def update(self):
    sql = """
        UPDATE employees
        SET name = ?, job_title = ?, department_id = ?
        WHERE id = ?
    """
    CURSOR.execute(sql, (self.name, self.job_title,
                    self.department.id, self.id))
    CONN.commit()


@classmethod
def create(cls, name, job_title, department):
    """ Initialize a new Employee object and save the object to the database """
    employee = Employee(name, job_title, department)
    employee.save()
    return employee
```

The final change to the `Employee` class is to update the `new_from_db` method
to include the department when instantiating the `Employee` class instance. We
use the foreign key stored in `row[3]` (i.e. the `department_id` column in the
"employees" table) to retrieve the associated `Department` class instance.

```py
@classmethod
def new_from_db(cls, row):
    """Initialize a new Employee object using the values from the table row."""
    department = Department.find_by_id(row[3])
    employee = cls(row[1], row[2], department)
    employee.id = row[0]
    return employee
```

## Updating the `Department` class to compute associated `Employee` instances

Since `Department` is not on the owning side of the relationship, we won't store
anything about employees in the "departments" table. But we might want to get a
list of all employees that work for a department. We'll add a method to
`Department` that accomplishes this task by querying the "employees" table for
rows that contain a foreign key value that matches the current department's id.
We'll import the `Employee` class within the function definition to avoid an
issue with circular imports (since the `Employee` class imports the `Department`
class as well).

```py
from config import CURSOR, CONN

class Department:

    # existing methods ...

    def employees(self):
        from employee import Employee
        sql = """
            SELECT * FROM employees
            WHERE department_id = ?
        """
        CURSOR.execute(sql, (self.id,),)

        rows = CURSOR.fetchall()
        return [
            Employee(row[1], row[2], self, row[0]) for row in rows
        ]
```

## Exploring the one-to-many relationship

The file `lib/debug.py` has been updated to initialize the database with two
departments and several employees. The code generates fake names for the
employees, and picks a random job title and department for each employee.

Run the file to recreate the database with sample data:

```bash
python lib/debug.py
```

In the `ipdb` session, you can explore the table data using the ORM methods
(your employee data will be different):

```py
ipdb> Department.get_all()
[<Department 1: Payroll, Building A, 5th Floor>, <Department 2: Human Resources, Building C, East Wing>]
```

```py
ipdb> Employee.get_all()
[<Employee 1: Deborah Diaz, Web Designer, Department: Payroll >, <Employee 2: Darlene Cook, Web Designer, Department: Human Resources >, <Employee 3: Paul Welch, Database Administrator, Department: Payroll >, <Employee 4: Matthew King, Full-stack Engineer, Department: Payroll >, <Employee 5: Tracey Cox, Manager, Department: Payroll >]
```

Let's select the payroll department:

```py
ipdb> payroll = Department.find_by_id(1)
ipdb> payroll
<Department 1: Payroll, Building A, 5th Floor>
```

We can call the `employees()` method to get the list of employees that work in
the payroll department.

```py
ipdb> payroll.employees()
[<Employee 1: Deborah Diaz, Web Designer, Department: Payroll >, <Employee 3: Paul Welch, Database Administrator, Department: Payroll >, <Employee 4: Matthew King, Full-stack Engineer, Department: Payroll >, <Employee 5: Tracey Cox, Manager, Department: Payroll >]
```

Let's try the other side of the relationship, getting the department associated
with a given employee:

```py
ipdb> employee = Employee.find_by_id(1)
ipdb> employee
<Employee 1: Deborah Diaz, Web Designer, Department: Payroll >
ipdb> employee.department
<Department 1: Payroll, Building A, 5th Floor>
```

You can also use the SQLITE EXPLORER to view the relationship (your data will be
different):

![employee table](https://curriculum-content.s3.amazonaws.com/7134/python-p3-v2-orm/employee_table.png)

## Testing the `Employee` and `Department` classes

The `lib/testing` folder contains the files `department_orm_test.py` and
`employee_orm_test.py` to test the `Department` and `Employee` classes. Run the
tests and confirm they pass:

```bash
pytest -x
```

## Conclusion

In this lesson we've seen how to persist a one-to-many relationship between
classes. The class or entity on the "many" side of the relationship is
responsible for storing and maintaining the relationship, since it only needs to
store a single value to reference the associated object/entity on the "one"
side.

---

## Solution Code

```py
from config import CURSOR, CONN


class Department:

    def __init__(self, name, location, id=None):
        self.id = id
        self.name = name
        self.location = location

    def __repr__(self):
        return f"<Department {self.id}: {self.name}, {self.location}>"

    @classmethod
    def create_table(cls):
        """ Create a new table to persist the attributes of Department class instances """
        sql = """
            CREATE TABLE IF NOT EXISTS departments (
            id INTEGER PRIMARY KEY,
            name TEXT,
            location TEXT)
        """
        CURSOR.execute(sql)
        CONN.commit()

    @classmethod
    def drop_table(cls):
        """ Drop the table that persists Department class instances """
        sql = """
            DROP TABLE IF EXISTS departments;
        """
        CURSOR.execute(sql)
        CONN.commit()

    def save(self):
        """ Insert a new row with the name and location values of the current Department object.
        Update object id attribute using the primary key value of new row"""
        sql = """
            INSERT INTO departments (name, location)
            VALUES (?, ?)
        """

        CURSOR.execute(sql, (self.name, self.location))
        CONN.commit()

        self.id = CURSOR.lastrowid

    @classmethod
    def create(cls, name, location):
        """ Initialize a new Department object and save the object to the database """
        department = Department(name, location)
        department.save()
        return department

    def update(self):
        """Update the table row corresponding to the current Department object."""
        sql = """
            UPDATE departments
            SET name = ?, location = ?
            WHERE id = ?
        """
        CURSOR.execute(sql, (self.name, self.location, self.id))
        CONN.commit()

    def delete(self):
        """Delete the table row corresponding to the current Department class instance"""
        sql = """
            DELETE FROM departments
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

    @classmethod
    def new_from_db(cls, row):
        """Return a new Department object using the values from the table row."""
        department = cls(row[1], row[2])
        department.id = row[0]
        return department

    @classmethod
    def get_all(cls):
        """Return a list containing a new Department object for each row in the table"""
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all

    @classmethod
    def find_by_id(cls, id):
        """Return a new Department object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM departments
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.new_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return a new Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None

    def employees(self):
        from employee import Employee
        sql = """
            SELECT * FROM employees
            WHERE department_id = ?
        """
        CURSOR.execute(sql, (self.id,),)

        rows = CURSOR.fetchall()
        return [
            Employee(row[1], row[2], self, row[0]) for row in rows
        ]
```

```py
from config import CURSOR, CONN
from department import Department


class Employee:

    def __init__(self, name, job_title, department, id=None):
        self.id = id
        self.name = name
        self.job_title = job_title
        self.department = department

    def __repr__(self):
        return (
            f"<Employee {self.id}: {self.name}, {self.job_title}, "
            + f"Department: {self.department.name} >")

    @classmethod
    def create_table(cls):
        sql = """
            CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY,
            name TEXT,
            job_title TEXT,
            department_id INTEGER,
            FOREIGN KEY (department_id) REFERENCES departments(id))
        """
        CURSOR.execute(sql)
        CONN.commit()

    @classmethod
    def drop_table(cls):
        """ Drop the table that persists Employee class instances """
        sql = """
            DROP TABLE IF EXISTS employees;
        """
        CURSOR.execute(sql)
        CONN.commit()

    def save(self):
        sql = """
            INSERT INTO employees (name, job_title, department_id)
            VALUES (?, ?, ?)
        """

        CURSOR.execute(sql, (self.name, self.job_title, self.department.id))
        CONN.commit()

        self.id = CURSOR.lastrowid

    def update(self):
        sql = """
            UPDATE employees
            SET name = ?, job_title = ?, department_id = ?
            WHERE id = ?
        """
        CURSOR.execute(sql, (self.name, self.job_title,
                             self.department.id, self.id))
        CONN.commit()

    @classmethod
    def create(cls, name, job_title, department):
        """ Initialize a new Employee object and save the object to the database """
        employee = Employee(name, job_title, department)
        employee.save()
        return employee

    def delete(self):
        sql = """
            DELETE FROM employees
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

    @classmethod
    def new_from_db(cls, row):
        """Initialize a new Employee object using the values from the table row."""
        department = Department.find_by_id(row[3])
        employee = cls(row[1], row[2], department)
        employee.id = row[0]
        return employee

    @classmethod
    def get_all(cls):
        """Return a list containing one Employee object per table row"""
        sql = """
            SELECT *
            FROM employees
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all

    @classmethod
    def find_by_id(cls, id):
        """Return Employee object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM employees
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.new_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return Employee object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM employees
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None

```
