# Discriminator Field Name Specs
Author: Neil Shweky  
Reviewer: Oleg Pudeyev

## Summary

This project is aimed at making it possible for users to name the discriminator field name used when having multiple collections that inherit from a single parent collection. 

## Motivation

Currently, when creating a document in a collection that inherits from another collection, a `_type` field will be created in the document to specify which of the child collections the document belongs to. I want to provide the user with the option to name this field to something other than `_type`, a feature provided in other ODM's like Mongoose and Doctrine.

## Goals 
- Create a user-facing API that allows the user to easily set the discriminator field name.
- Implement a way to specify the discriminator field name

## ActiveRecord Example
The following is an example of how the discriminator field name (called the inheritance_column) is specified in Mongoose:
```ruby
# shape.rb
class Shape < ApplicationRecord
    self.inheritance_column = "type2"
end

# circle.rb
class Circle < Shape
end

# 20200709125447_create_shapes.rb
class CreateShapes < ActiveRecord::Migration[6.0]
  def change
    create_table :shapes do |t|
      t.numeric :x
      t.numeric :y
      t.string :type2

      t.timestamps
    end
  end
end


# seeds.rb
circle = Circle.create(x: 1, y:2)

# Example Circle row
id|x|y|type2 |        created_at        |        updated_at       |
1 |1|2|Circle|2020-07-09 13:03:00.201087|2020-07-09 13:03:00.201087
```


## Mongoose Example 

The following is an example of how the discriminator field name (called discriminatorKey) is specified in Mongoose:
```js
const { Schema } = mongoose

var Shape = mongoose.model('Shape', new Schema({
  x: Number,
  y: Number
}, {
  discriminatorKey: 'shape_type',
  collection: 'shapes'
}));

var Circle = Shape.discriminator('Circle', new Schema({}));
new Circle().save()

var rectangleSchema = new Schema({});
var Rectangle = Shape.discriminator('Rectangle', rectangleSchema, 'rect');
new Rectangle().save()

// Example Circle document
{
    "_id": ObjectId("5f06232bde3b7d13721e17f3"),
    "shape_type": "Circle",
    "__v": 0
}

// Example Rectangle document
{
    "_id": ObjectId("5f06232bde3b7d13721e17f4"),
    "shape_type": "rect",
    "__v": 0
}
```
As you can see, the discriminator field name is specified in the parent schema. Mongoose also has the option to specify the value of the discriminator for the child schemas. This is why the `shape_type` in the Rectangle document has "rect" as its value, as was set in the third parameter to the `Shape.discriminator()` function.

## Doctrine Example: 
The following is an example of how the discriminator field name (called the DiscriminatorColumn) is specified in Doctrine:
```php
<?php
namespace MyProject\Model;

/**
 * @Entity
 * @InheritanceType("SINGLE_TABLE")
 * @DiscriminatorColumn(name="discr", type="string")
 * @DiscriminatorMap({"shape" = "Shape", "circle" = "Circle"})
 */
class Shape
{
    // ...
}

/**
 * @Entity
 */
class Circle extends Shape
{
    // ...
}
```

## Proposed Functionality

The proposed functionality, and the way it works in Mongoose, is as follows:

- The user will be able to set the discriminator field name in the class of the parent collection.
- On creation of document, a field will be created with the discriminator field name that the user specified in the parent class, with the value of the class name as a string.
- If the discriminator field name is changed after documents have been added to the collection, the new field name will be used for all future documents while leaving the old ones unmodified.

## User-Facing API

The user will be able to specify the discriminator field name in the base class, as follows:

```ruby
class Shape
    include Mongoid::Document
    field :x, type: Integer
    field :y, type: Integer

    discriminator_field_name = :shape_type
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
There is an added `discriminator_field_name` attribute that I will be adding.

## Implementation Plan

The following are my plans for the implementation of this feature:

- The main code for adding the `_type` field is in `traversable.rb` in a function titled `inherited`. There are also a few other places where `:type` are hard-coded that I have to take care of.
- Add an `attr_accessor` to the `travesable.rb` file that holds the discriminator field name. This variable should default to `:_type`.
- Change the `inherited` function to use this variable instead of hard-coding `:type`.
- Modify the `criteria.rb` file to change the hard-coded `:type` to use this attribute to get the discriminator field name.

## Assumptions and Risks

The biggest risk of this project is that there might be a piece of code that relies on the discriminator field name being `_type`, and the code breaks because of it.

## Dependencies
None

## Open Questions
- Should the function be called `discriminator_key` or `inheritance_field`? **discriminator_field_name**
- Should we use a function for setting the discriminator or an assignment, like ActiveRecord does? **assignment, attr_accessor**

## Complexity Estimate
I think this project should take about 1-2 weeks to complete. This project cannot be parallelized between multiple engineers.

## Future
A potential added feature to this project would be allowing the user to specify the value of the discriminator for each child class. Right now, the default value is the class as a string. The suggested field name is dricriminator_mapping.


## Testing

None of the current tests should break as a result of this implementation, so it should be helpful to look at the current tests to make sure I haven't broken anything. In terms of testing the new feature, here are some things I would like to test: 

1. One subclass: A case where there is one parent and one child. 
2. Multiple subclasses: When there is a parent and multiple children.
3. When changing the discriminator field name after documents have been added.