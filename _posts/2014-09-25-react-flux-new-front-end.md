---
layout:     post
title:      A RefluxJS Based Rest Store
date:       2014-09-25 12:31:19
summary:    Routers and REST based stores for RefluxJS
categories: flux react javascript
---

Kicking over some convention over configuration stuff, I've put together a rest based extension to [RefluxJS](https://github.com/spoike/refluxjs) stores.

This is how it works:

{% highlight coffeescript %}
    RefluxRestStore.createCollectionStore "TestStores.Chaos",
      path: "/api/{kallistiId}/chaos"
{% endhighlight %}

This will automatically create a store called `TestStores.Chaos.CollectionStore` and the action `TestActions.Chaos.loadCollection`. invoking `loadCollection` will cause the store to trigger with the collection and its sideloaded data. An excerpt from the spex

{% highlight elixir %}

  it "should listenTo the loadCollection and call it with the correct url", ->
    @server.respondWith JSON.stringify(Mocks.CollectionResponse)
    listener = jasmine.createSpy "collectionResponse"

    TestStores.Chaos.CollectionStore.listen listener
    TestActions.Chaos.loadCollection kallistiId: "KALLISTI"

    jasmine.Clock.tick 1
    @server.respond()

    expect(listener).toHaveBeenCalled()
    response = listener.mostRecentCall.args[0]

    expect(response.loading).toBeFalsy()
    expect(response.isError).toBeFalsy()
    expect(response.newObjects.length).toEqual Mocks.CollectionResponse.chaos.length
    expect(response.newObjects[0].forum.id).toEqual response.newObjects[0].forum_id

{% endhighlight %}

Similar stuff werks for the `createObjectStore` method. Here's the source

{% highlight coffeescript %}
pkg "RefluxRestStore", ->
  _sideload = (payload, resource) ->
    _.each resource, (value, attribute) ->
      if matched = attribute.match(/(.+)_id$/)
        resource[matched[1]] = _.findWhere payload[matched[1].pluralize()], id: resource[attribute]

  exports =
    _createStore: (kind, namespace, definition) ->
      pkg "#{namespace}.#{kind}Store", ->
        actionNamespace           = namespace.replace /Stores/, 'Actions'
        resourceName              = _.str.underscored _.last(actionNamespace.split("."))
        actionName                = "load#{kind}"
        actionPackage             = getPkg actionNamespace
        actionPackage[actionName] = Reflux.createAction actionName
        action                    = actionPackage["load#{kind}"]
        init                      = definition.init || (->)

        definition = _.extend definition,
          init: ->
            @resourceName = resourceName
            @listenTo action, @["onLoad#{kind}"].bind(@)
            init.call this

        Reflux.createStore definition

    createObjectStore: (namespace, definition) ->
      @_createStore "Object", namespace, _.extend definition,
        onLoadObject: (params) ->
          params ?= { }
          url     = pathForPattern @path, params
          params  = _.omit params, pathKeys(@path)
          @trigger loading: true
          promise = $.ajax
            url: url
            data: params
            type: 'GET'
            dataType: 'json'

          promise.success (payload) =>
            _sideload payload, payload[@resourceName]
            @trigger loading: false, object: payload[@resourceName]
          promise.fail(error) =>
            @trigger loading: false, isError: true, errors: errors

    createCollectionStore: (namespace, definition) ->
      @_createStore "Collection", namespace, _.extend definition,
          onLoadCollection: (params) ->
            params ?= { }
            url     = pathForPattern @path, params
            params  = _.omit params, pathKeys(@path)

            @trigger loading: true
            promise = $.ajax
              url: url
              data: params
              type: 'GET'
              dataType: 'json'

            promise.success (payload) =>
              _.each payload[@resourceName], (resource) =>
                _sideload payload, resource
              @trigger loading: false, newObjects: payload[@resourceName]

            promise.fail (errors) =>
              @trigger loading: false, isError: true, errors: errors

{% endhighlight %}
