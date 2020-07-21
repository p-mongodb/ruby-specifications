# Discriminator Map Specs
Author: Neil Shweky  
Reviewer: Oleg Pudeyev

## Summary

This project is aimed at making it possible for users to override the mapping for the discriminator key for a child class.

## Motivation

Currently, when creating a document in a collection that inherits from another collection, a discriminator field is created, with the key defaulting to `_type` (which we can now successfully override with MONGOID-4817), and the value as the class as a string. The goal of this project is to give the user the ability to override the default value of this field.

## Goals 
- Create a user-facing API that allows the user to easily set the discriminator key.
- Implement a way to specify the discriminator key

## ActiveRecord Example
The following is an example of how the discriminator key (called the inheritance_column) is specified in ActiveRecord:
```ruby
# shape.rb
class Shape < ApplicationRecord
    self.inheritance_column = "type2"
end

# circle.rb
class Circle < Shape
  def self.sti_name
    "Round Thing"
  end
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
id|x|y|   type2   |        created_at        |        updated_at       |
11|1|2|Round Thing|2020-07-21 15:34:55.148343|2020-07-21 15:34:55.148343
```

## Mongoose Example 

The following is an example of how the discriminator key (called discriminatorKey) is specified in Mongoose:
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
As you can see, the discriminator key is specified in the parent schema. Mongoose also has the option to specify the value of the discriminator for the child schemas. This is why the `shape_type` in the Rectangle document has `rect` as its value, as was set in the third parameter to the `Shape.discriminator()` function.

## Doctrine Example: 
The following is an example of how the discriminator key (called the DiscriminatorColumn) is specified in Doctrine:
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

- The user will be able to set the discriminator value in each child class.
- On creation of document, a field will be created with the discriminator key that the user specified in the parent class, with the value of the class name as a string.
- If the discriminator key is changed after documents have been added to the collection, the new key will be used for all future documents while leaving the old ones unmodified.
- The subclass should not be able to overwrite the dicriminator key of the parent class.

## User-Facing API

The user will be able to specify the discriminator value in the child classes, as follows:

```ruby
class Shape
    include Mongoid::Document
    field :x, type: Integer
    field :y, type: Integer
end

class Circle < Shape
    field :radius, type: Float

    self.discriminator_mapping = "Round Thing"
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
    "_type" : "Round Thing", 
    "radius" : 3 
}

# Example Rectangle Document
{
    "_id": ObjectId("5f0602261a819b0e0cbc2e5c"),
    "_type": "Rectangle",
    "width": 2,
    "height": 1
}
```
There is an added `discriminator_mapping` attribute that I will be adding.

## Implementation Plan

The following are my plans for the implementation of this feature:

The main code for adding the discriminator value is in `traversable.rb` in a function titled `inherited`. There are also a few other places where `class.to_s` is hard-coded that have to be changed.

1. Add a class variable to `traversable.rb` that is a `class_attribute` and defaults to `class.to_s`.
2. Prepend the `discriminator_mapping` function with the hereditary check, to make it unable to set the `discriminator_mapping` from the parent.
3. Change the `inherited` function to use this variable inside the `default_proc`.
4. Modify the `criteria.rb` (and other) files to change the hard-coded `class.to_s` to use the `discriminator_mapping` class variable to get the discriminator value.

## Assumptions and Risks
None

## Dependencies

Mongoid-4817: Overriding the default disciminator key.

## Open Questions
- Should the function be called `discriminator_mapping` or `discriminator_value`?
- Should we use a function for setting the discriminator or an assignment, like ActiveRecord does?

## Complexity Estimate
I think this project should take about 1-2 weeks to complete. This project cannot be parallelized between multiple engineers.

## Future
None


## Testing

None of the current tests should break as a result of this implementation, so it should be helpful to look at the current tests to make sure I haven't broken anything. In terms of testing the new feature, here are some things I would like to test: 

1. One subclass: A case where there is one parent and one child. 
2. Multiple subclasses: When there is a parent and multiple children.
3. When changing the discriminator value after documents have been added.
6. Attempting to set from the parent and expecting it to fail.
