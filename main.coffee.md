Model
=====

The `Model` module provides helper methods to compose nested data models.

Models uses [Observable](/observable/docs) to keep the internal data in sync.

    Core = require "./core"
    Observable = global.Observable ? require "observable"

    module.exports = Model = (I={}, self=Core(I)) ->

      self.extend

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

    {defaults, extend} = require "./util"
    Object.assign Model, {Core, Observable, defaults, extend}
