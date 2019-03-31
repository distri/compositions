Model
=====

The `Model` module provides helper methods to compose nested data models.

Models uses [Observable](/observable/docs) to keep the internal data in sync.

    Observable = global.Observable ? require "observable"

    module.exports = Model = (I={}, self={}) ->

      Object.assign self,

External access to instance variables. Use of this property should be avoided
in general, but can come in handy from time to time.

>     #! example
>     I =
>       r: 255
>       g: 0
>       b: 100
>
>     myObject = Model(I)
>
>     [myObject.I.r, myObject.I.g, myObject.I.b]

        I: I

Generates a public jQuery style getter / setter method for each `String` argument.

>     #! example
>     myObject = Model
>       r: 255
>       g: 0
>       b: 100
>
>     myObject.attrAccessor "r", "g", "b"
>
>     myObject.r(254)

        attrAccessor: (attrNames...) ->
          attrNames.forEach (attrName) ->
            self[attrName] = (newValue) ->
              if arguments.length > 0
                I[attrName] = newValue

                return self
              else
                I[attrName]

          return self

Generates a public jQuery style getter method for each String argument.

>     #! example
>     myObject = Model
>       r: 255
>       g: 0
>       b: 100
>
>     myObject.attrReader "r", "g", "b"
>
>     [myObject.r(), myObject.g(), myObject.b()]

        attrReader: (attrNames...) ->
          attrNames.forEach (attrName) ->
            self[attrName] = ->
              I[attrName]

          return self

Extends this object with methods from the passed in object. A shortcut for Object.extend(self, methods)

>     I =
>       x: 30
>       y: 40
>       maxSpeed: 5
>
>     # we are using extend to give player
>     # additional methods that Model doesn't have
>     player = Model(I).extend
>       increaseSpeed: ->
>         I.maxSpeed += 1
>
>     player.increaseSpeed()

        extend: (objects...) ->
          Object.assign self, objects...

Includes a module in this object. A module is a constructor that takes two parameters, `I` and `self`

>     myObject = Model()
>     myObject.include(Bindable)

>     # now you can bind handlers to functions
>     myObject.bind "someEvent", ->
>       alert("wow. that was easy.")

        include: (modules...) ->
          for Module in modules
            Module(I, self)

          return self


Bind a data model getter/setter to an attribute. The data model is bound directly to
the attribute and must be directly convertible to and from JSON.

        attrData: (name, DataModel) ->
          I[name] = DataModel(I[name])

          Object.defineProperty self, name,
            get: ->
              I[name]
            set: (value) ->
              I[name] = DataModel(value)

Observe any number of attributes as observables. For each attribute name passed in we expose a public getter/setter method and listen to changes when the value is set.

        attrObservable: (names...) ->
          names.forEach (name) ->
            self[name] = Observable(I[name])

            self[name].observe (newValue) ->
              I[name] = newValue

          return self

Observe an attribute as a model. Treats the attribute given as an Observable
model instance exposing a getter/setter method of the same name. The Model
constructor must be passed explicitly.

        attrModel: (name, Model) ->
          model = Model(I[name])

          self[name] = Observable(model)

          self[name].observe (newValue) ->
            I[name] = newValue.I

          return self

Observe an attribute as an array of sub-models. This is the same as `attrModel`
except the attribute is expected to be an array of models rather than a single one.

        attrModels: (name, Model) ->
          models = (I[name] or []).map (x) ->
            Model(x)

          self[name] = Observable(models)

          self[name].observe (newValue) ->
            I[name] = newValue.map (instance) ->
              instance.I

          return self

Delegate methods to another target. Makes it easier to compose rather than extend.

        delegate: (names..., {to}) ->
          names.forEach (name) ->
            console.log "delegating #{name} to #{to}"
            Object.defineProperty self, name,
              get: ->
                receiver = getValue self, to
                receiver[name]
              set: (value) ->
                receiver = getValue self, to
                setValue receiver, name, value

The JSON representation is kept up to date via the observable properites and resides in `I`.

        toJSON: ->
          I

Return our public object.

      return self

Helper functions:

    isFn = (x) ->
      typeof x is 'function'

    getValue = (receiver, property) ->
      if isFn receiver[property]
        receiver[property]()
      else
        receiver[property]

    setValue = (receiver, property, value) ->
      target = receiver[property]

      if isFn target
        target.call(receiver, value)
      else
        receiver[property] = value

    defaults = (target, objects...) ->
      for object in objects
        for name of object
          unless target.hasOwnProperty(name)
            target[name] = object[name]

      return target

    extend = Object.assign

    Object.assign Model, {Observable, defaults, extend}
