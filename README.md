# Discriminator Key Specs
Author: Neil Shweky  
Reviewer: Oleg Pudeyev

## Summary

This project is aimed at making it possible for users to name the discriminator key used when having multiple collections that inherit from a single parent collection. 

## Motivation

Currently, when creating a document in a collection that inherits from another collection, a `_type` field will be created in the document to specify which of the child collections the document belongs to. I want to provide the user with the option to name this field to something other than `_type`, a feature provided in other ODM's like Mongoose and Doctrine.

## Goals 
- Implement a way to specify the discriminator key
- Create a user-facing API that allows the user to easily set the discriminator key.

## Mongoose example 

The following is an example of how the discriminator key is specified in Mongoose:
```js
const { Schema } = mongoose

var Person = mongoose.model('Person', new Schema({
  name: String,
  createdAt: Date
}, {
  discriminatorKey: 'emp_types',
  collection: 'people'
}));

var Boss = Person.discriminator('Boss', new Schema({}));
new Boss().save()

var employeeSchema = new Schema({ boss: String });
var Employee = Person.discriminator('Employee', employeeSchema, 'staff');
new Employee().save()

// Example Boss document
{
    "_id": ObjectId("5f04b51b47fbfa7351a8b040"),
    "emp_type": "Boss",
    "__v": 0
}

// Example Employee document
{
    "_id": ObjectId("5f04b51b47fbfa7351a8b041"),
    "emp_type": "staff",
    "__v": 0
}
```
As you can see, the discriminator key is specified in the parent schema. Mongoose also has the option to specify the value of the discriminator for the child schemas. This is why the `emp_type` in the employee document has "staff" as its value, as was set in the third parameter to the `Person.discriminator()` function.

For an example using PHP and Doctrine, go to the end of this document.

## Proposed Functionality

The proposed functionality, and the way it works in Mongoose, is as follows:

- The user will be able to set the discriminator key in the class of the parent collection.
- On creation of document, a field will be created with the discriminator key that the user specified in the parent class, with the value of the class name as a string.
- If the discriminator key is changed after documents have been added to the collection, the new key will be used for all future documents while leaving the old ones unmodified.

## User-Facing API

The user will be able to specify the discriminator key in the base class, as follows:

```ruby
class Shape
    include Mongoid::Document
    field :x, type: Integer
    field :y, type: Integer

    discriminator_key :shape_type
end

class Circle < Shape
    field :radius, type: Float
end

class Rectangle < Shape
    field :width, type: Float
    field :height, type: Float
end

Circle.create!(radius: 3)
Rectangle.create!(width: 2, height: 1)

# Example Circle document
{
    "_id" : ObjectId("5f05c6fe1a819b742567379a"), 
    "shape_type" : "Circle", 
    "radius" : 3 
}

# Example Rectangle Document
{
    "_id": ObjectId("5f0602261a819b0e0cbc2e5c"),
    "shape_type": "Rectangle",
    "width": 2,
    "height": 1
}
```
There is an added `discriminator_key` function that I will be adding.

## Implementation Plan

The following are my plans for the implementation of this feature:

- The main code for adding the `_type` field is in `traversable.rb` in a function titled `inherited`. There are also a few other places where `:type` are hard-coded that I have to take care of.
- Add an `attr_reader` to the `travesable.rb` file that holds the discriminator key. This variable should default to `_:type`.
- Change the `inherited` function to use this variable instead of hard-coding `:type`.
- Create a function in `traversable.rb` that returns the discriminator key.
- Modify the `criteria.rb` file to change the hard-coded `:type` to use this function to get the discriminator key
- Open the `ClassMethods` module in `traversable.rb` file and create the `discriminator_key` function that sets the `attr_reader`.

## Assumptions and Risks

The biggest risk of this project is that there might be a piece of code that relies on the discriminator key being `_type`, and the code breaks because of it.

## Dependencies
None

## Open Questions
None

## Complexity Estimate
I think this project should take about 1-2 weeks to complete. This project cannot be parallelized between multiple engineers.

## Future
A potential added feature to this project would be allowing the user to specify the value of the discriminator for each child class. Right now, the default value is the class as a string.

## Doctrine Example: 
```php
<?php
namespace MyProject\Model;

/**
 * @Entity
 * @InheritanceType("SINGLE_TABLE")
 * @DiscriminatorColumn(name="discr", type="string")
 * @DiscriminatorMap({"person" = "Person", "employee" = "Employee"})
 */
class Person
{
    // ...
}

/**
 * @Entity
 */
class Employee extends Person
{
    // ...
}
```