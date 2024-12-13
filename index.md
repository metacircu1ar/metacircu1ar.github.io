---
layout: post
title: "Implementing a simple object system from scratch in Ruby"
date: 2024-12-14
---
> "What I cannot create, I do not understand." -- Richard Feynman

What I found out over time, is that the best way to really learn something is to implement it yourself. What we're going to do is implement an object system from scratch. We won't be using classes and just build it from anonymous functions and hash tables. We'll implement method lookup, prototypical inheritance, mixins and some form of metaprogramming. I'm going to use Ruby for this, because it's one of my favorite languages and it's very expressive and concise.

Let's start with writing a constructor function. It takes some arguments, creates an object and returns it. Let's call this function `initialize`. It will return a hash object, which will represent an instance of some hypothetical class. For now, it will return an empty hash, which represents an object without methods.
```ruby
def initialize
  {
  }
end
```
We will have constructors for objects of different classes, so we need to organize our code in some way. Let's create a module, which will serve as a namespace for our constructor and other utility functions related to our class. We'll name the module the same way we would name our class. For example:
```ruby
# We define a module which will be responsible for storing 
# the constructor function for the Entity class.
module Entity
  # Constructor function for the Entity class.
  def self.initialize
    {
    }
  end
end

# Module for storing the Player class related functions.
module Player
  # Constructor function for the Player class.
  def self.initialize
    {
    }
  end
end
```
We'll use hash keys as method names and anonymous functions as methods. We could use strings(such as `"greet"`) as method names, but Ruby has symbols(such as `:greet`) and we'll use those instead, because it's a bit cleaner. We can define a method like this:
```ruby
module GreetingClass
  def self.initialize
  # Returns a hash with a single key-value pair, 
  # where the key is a symbol :greet and the value 
  # is an anonymous function which prints "Hello" when called.
    {
      :greet => -> { puts "Hello" }
    }
  end
end

# We can create an object of our class by calling 
# the constructor.
greeting_object = GreetingClass.initialize
# We can call the method on our object by looking up 
# the key in the hash table and calling the anonymous 
# function which is the value of that key.
greeting_object[:greet].call
```
All local variables we declare in the constructor have private visibility by default:
```ruby
module GreetingClass
  def self.initialize
    # Private instance variable of our object invisible 
    # from the outside, unless we expose it using methods.
    message = "Hello"
    {
      # We can access the message variable inside this 
      # anonymous function, because it's a closure.
      :greet => -> { puts message } 
    }
  end
end
```
The public interface of our object is only exposed through the methods we define on the object. We can define methods on the object by adding them as keys to the hash table we return from the constructor. For example, let's create a public setter for the message:
```ruby
module GreetingClass
  def self.initialize
    message = "Hello"
    {
      :greet => -> { puts message },
      # Updates the message variable.
      :set_message => ->(new_message) { message = new_message }
    }
  end
end

greeting_object = GreetingClass.initialize
puts greeting_object[:greet].call # => Hello

# Sets the message to "Hi"
greeting_object[:set_message].call("Hi") 
puts greeting_object[:greet].call # => Hi
```
Note, that we can't access the `message` variable directly from the outside of our constructor function, because it's a local variable in the constructor. We can only access it through the methods we define on the object. Anonmyous functions we store in the hash object are closures, so they capture the environment in which they were defined. That's why the `greet` and `:set_message` methods can access the `message` variable, even though it's not passed as an argument to these methods directly. This way we can encapsulate the state of our object and expose only the methods we want to be public. When we change the state of the object, this change is reflected in all methods, because they all share the same environment in which the closures were defined, so they all share a reference to the same `message` variable. We just simulated encapsulation and instance variables.

Let's create one more simple class as an example of our approach:

```ruby
# Define a module for demonstrating arithmetic operations.
module ArithmeticExampleClass
  # This class exposes arithmetic methods like add, 
  # multiply, and setters for operands a and b.
  def self.initialize(a, b)
    # Encapsulate 'a' and 'b' variables.
    # a and b represent internal state of our object.
    {
      add: -> { a + b },              # Adds a and b.
      multiply: -> { a * b },         # Multiplies a and b.
      set_a: ->(new_a) { a = new_a }, # Setter for a.
      set_b: ->(new_b) { b = new_b }  # Setter for b.
    }
  end
end

# Create an instance of ArithmeticExampleClass with initial 
# values for a and b.
object = ArithmeticExampleClass.initialize(3, 4)

# Call arithmetic methods to demonstrate functionality.
puts "Addition result: #{object[:add].call}" # => 7
puts "Multiplication result: #{object[:multiply].call}" # => 12

# Update values of a and b to demonstrate mutable setters.
object[:set_a].call(10)
object[:set_b].call(20)

# Call methods again with updated values.
puts "Addition result: #{object[:add].call}" # => 30
puts "Multiplication result: #{object[:multiply].call}" # => 200
```

In class-based OOP we use classes, which delegate behavior to each other using inheritance. Inheritance is a fixed form of delegation. We don't have classes, but we still want to achieve delegation, so what we do instead is link our objects using prototypes. Instead of class A inheriting from class B, we'll have object B set as a prototype of object A.

Built-in Ruby hash class is insufficient for our needs, because its `[]` method only looks up methods only in the hash itself, but we also want to look up methods in the prototype object. We can create our own hash class which will delegate the method lookup to the prototype object if the method is not found in current object.

```ruby
# Define a HashObject class for prototype-based 
# inheritance and method lookup.
# We base it on built-in Ruby Hash class and 
# customize some operations for our needs.
class HashObject < Hash
  def initialize(initial_hash)
    self.merge!(initial_hash)
  end
  # Override the [] method to allow prototype-based 
  # method lookup.
  # This [] method first looks for the key(method name) 
  # in the current object. If the method is found, it returns 
  # the function associated with the method.
  # If the method is not found, it checks the :__prototype 
  # object for the method.
  # :__prototype key links to the prototype object, which was 
  # set using the 'inherit' method below.
  # The method lookup continues recursively until the method 
  # is found or the chain ends.
  def [](method)
    super(method) || super(:__prototype)&.[](method)
  end

  # Links a prototype object to the current object by 
  # setting the :__prototype key.
  # This allows the current object to delegate method 
  # lookups to the prototype.
  def inherit(prototype_object)
    self[:__prototype] = prototype_object
    self
  end

  # Explicitly fetches a method or key from the 
  # prototype without looking in the current object.
  # The method looks for the key in the prototype 
  # object (if it exists) and returns its value.
  # See the example below for usage.
  def proto(method)
    self[:__prototype]&.[](method)
  end
end
```
To put it another way, we just create a singly linked list of objects, where each object has a reference to the next object in the chain. When we look up a method on an object, we first look in the object itself, and if the method is not found, we look in the prototype object. If the method is not found in the prototype object, we look in the prototype of the prototype and so on, until we find the method or reach the end of the chain.

Let's build something more complex. Here are some module definitions:
```ruby
# Define a module Entity that serves as a base 
# class for other objects.
# This is a generic entity that provides methods 
# to get and set the name.
module Entity
  def self.initialize
    # Encapsulate the 'name' variable to simulate 
    # private instance-like behavior.
    name = "Generic entity"
    HashObject.new({
      # Getter for the name.
      get_name: -> { name }, 
      # Setter for the name.
      set_name: ->(new_name) { name = new_name } 
    })
  end
end

# Define a Player module, which will be able 
# to create Player objects.
# Player extends Entity by modifying or adding 
# additional functionality.
module Player
  def self.initialize
    # 'itself' variable is a reference to the 
    # object itself, allowing access to the object 
    # inside its own methods, like 'get_name' method 
    # below. This is useful for self-referential calls.
    itself = HashObject.new({
      # The get_name method appends " (player)" to 
      # the name from the prototype (Entity).
      # itself.proto(:get_name) is similar to super() 
      # method call in Ruby.
      get_name: -> { itself.proto(:get_name).call + " (player)" } 
    })
  end
end

# Define a Position module for objects that can 
# have a 2D position.
# This module manages the x and y coordinates 
# of the object.
module Position
  def self.initialize(x, y)
    # Encapsulate 'x' and 'y' variables to act 
    # as private state.
    HashObject.new({
      # Getter for x-coordinate.
      get_x: -> { x },
      # Getter for y-coordinate.
      get_y: -> { y }, 
      # Setter for x-coordinate.
      set_x: ->(new_x) { x = new_x }, 
      # Setter for y-coordinate.
      set_y: ->(new_y) { y = new_y }
    })
  end
end
```
And here are some usage examples:
```ruby
# Create an Entity instance.
entity = Entity.initialize

# Create a Player instance and inherit methods 
# from the Entity prototype.
# Inheritance means Player will be able to access 
# methods defined in Entity.
player = Player.initialize.inherit(entity)

# Test name functionality in Player by calling 
# inherited 'get_name' method.
puts "Player name: #{player[:get_name].call}" # => "Generic entity (player)"
player[:set_name].call("John") # Update name via Player's setter.
puts "Player name: #{player[:get_name].call}" # => "John (player)"

# Now create an inheritance chain: 
# Player -> Entity -> Position.
# Entity inherits from Position and Player 
# inherits from Entity.
# This allows Player to inherit methods 
# from both Entity and Position.
# Prototype method lookup will traverse the 
# inheritance chain to find methods.
player = Player.initialize
entity = Entity.initialize
position = Position.initialize(10, 20)

# Chain inheritance by making Entity inherit 
# Position, and then Player inherits Entity.
entity.inherit(position)
player.inherit(entity)

# Access methods through the inheritance chain, 
# now Player has access to Position's methods.
puts "Player position via inheritance chain: (#{player[:get_x].call}, #{player[:get_y].call})" # => (10, 20)
```
We can very easily implement mixins, because our objects are just hash tables. We can merge two hash tables together to mix in methods from one object into another.
```ruby
class HashObject < Hash
  # Mixes in another object methods into the 
  # current object.
  # This allows the extension of an object 
  # with additional methods.
  def mixin(object)
    self.merge!(object)
  end
end
```
Usage example:
```ruby
player = Player.initialize

# Create a Position instance with coordinates (10, 20).
position = Position.initialize(10, 20)

# Mix in the Position object into Player to add 
# position-related methods.
player.mixin(position)

# Access mixed-in position methods (get_x and get_y) 
# for the Player instance.
puts "Player position via mixin: (#{player[:get_x].call}, #{player[:get_y].call})" # => (10, 20)
```
As a final example, let's add some form of metaprogramming to our object system. We'll use metaprogramming to dynamically create methods for our objects. Let's create a version of Ruby's `attr_accessor` method, which generates getter and setter methods for attributes. For example, given the `:name` attribute, we will generate `get_name` and `set_name` methods for our object.
```ruby
class HashObject < Hash
  # Define a method to add attr_accessor functionality.
  def attr_accessor(*attributes)
    attributes.each do |attr|
      state = nil
      self.merge!({
        "get_#{attr}".to_sym => -> { state },
        "set_#{attr}".to_sym => ->(new_value) { state = new_value }
      })
    end
  end
end
```
Usage example:
```ruby
player = Player.initialize

# Generate getter and setter methods for 
# health and strength attributes.
player.attr_accessor(:health, :strength) 
player[:set_health].call(100)  # Set health to 100.
player[:set_strength].call(50)  # Set strength to 50.
puts "Player health: #{player[:get_health].call}"  # => 100
puts "Player strength: #{player[:get_strength].call}" # => 50
```

Here is a [link](https://gist.github.com/metacircu1ar/ce887fef160338f763a6a79c99b85c41) to the complete code listing with examples and comments.
