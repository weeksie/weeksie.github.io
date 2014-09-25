---
layout:     post
title:      A simple router for Reflux and ReactJS
date:       2014-09-26 12:31:19
summary:    Routers and REST based stores for RefluxJS
categories: flux react javascript router crossroads
---

So I've been experimenting with RefluxJS. React is purely a view component and Reflux uses the Flux pattern outlined by Facebook for building React applications. Obviously for a single page site, a router is nice to have. Here's a barebones router that's backed by crossroads.

{% highlight coffeescript %}
pkg "Router", ->
  class Router
    # TODO: interpolate this
    pageNamespace: "Components"
    constructor:(options) ->
      @routes = { }
      _(options).each (v, k) =>
        @[k] = v

    reset: ->
      @routes = { }
      crossroads.removeAllRoutes()

    draw:(mappingFunction) ->
      mappingFunction.call @

    resource: (resource, rest...) ->
      arg2 = rest[0]
      arg3 = rest[1]

      if _.isFunction arg2
        childF  = arg2
        options = { }
      else if _.isFunction arg3
        childF  = arg3
        options = arg2
      else if _.isObject arg2
        options = arg2
      else
        options = { }

      @routes = _.extend @routes, @_generateRoutes resource, options, childF

    _generateRoutes: (resource, options, childF) ->
      segments  = _ resource.split "."
      routes    = { }
      basePath  = options.path ? @_segmentsToPath(segments)
      basePath  = if @context? then "#{@context}#{basePath}" else basePath

      [ "index", "show", "edit" ].forEach (route) =>
        isIndex        = route == "index"
        token          = if isIndex then resource else "#{resource}.#{route}"
        routeSegments  = segments.concat route

        switch route
          when "index" then pattern = basePath
          when "show"  then pattern = "#{basePath}/{id}"
          when "edit"  then pattern = "#{basePath}/{id}/edit"


        routes[token] =
          page: @_segmentsToComponent routeSegments
          path: pattern
          route: crossroads.addRoute pattern, (args...) ->
            Actions.Routes.navigate routes[token], args

      if childF
        childRouter = new Router(context: "#{basePath}/{#{resource}Id}", pageNamespace: @pageNamespace)
        childF.call childRouter
        _.each childRouter.routes, (value, tok) ->
          routes["#{resource}.#{tok}"] = value

      routes

    _segmentsToComponent: (segments) ->
      segments.reduce ((acc, segment) =>
        acc[_.str.capitalize(segment)]
      ), window[@pageNamespace]

    _segmentsToPath: (segments) ->
      dasherized = segments.map (segment) -> _.str.dasherize(segment)
      "/#{dasherized.join('/'
{% endhighlight %}

The unit test to explain it:

{% highlight coffeescript %}
describe "Router", ->
  beforeEach ->
    container = document.createElement 'div'
    container.setAttribute 'id', 'app-root'
    document.documentElement.appendChild container
    @router = new Router(pageNamespace: "MockComponents")
    window.MockComponents =
      ChaosMachines:
        Index: "ChaosMachines.Index"
        Show: "ChaosMachines.Show"
        Edit: "ChaosMachines.Edit"

      Kallisti:
        Index: "Kallisti.Index"
        Show: "Kallisti.Show"

      Eris:
        Index: "Eris.Index"
      GoldenApple:
        Index: "GoldenApple.Index"

  afterEach ->
    container = document.getElementById 'app-root'
    container.parentNode.removeChild container
    @router.reset()

  it "should generate resource routes", ->
    @router.draw ->
      @resource "chaosMachines"

    expect(@router.routes["chaosMachines"].page).toEqual MockComponents.ChaosMachines.Index
    expect(@router.routes["chaosMachines.show"].page).toEqual MockComponents.ChaosMachines.Show
    expect(@router.routes["chaosMachines.edit"].page).toEqual MockComponents.ChaosMachines.Edit
    expect(@router.routes["chaosMachines"].path).toEqual "/chaos-machines"
    expect(@router.routes["chaosMachines.show"].path).toEqual "/chaos-machines/{id}"
    expect(@router.routes["chaosMachines.edit"].path).toEqual "/chaos-machines/{id}/edit"

  it "should generate nested routes", ->
    @router.draw ->
      @resource "chaosMachines", ->
        @resource "kallisti", ->
          @resource "goldenApple"
          @resource "eris"

    expect(@router.routes["chaosMachines.kallisti"].path).
      toEqual "/chaos-machines/{chaosMachinesId}/kallisti"

    expect(@router.routes["chaosMachines.kallisti.goldenApple"].path).
      toEqual "/chaos-machines/{chaosMachinesId}/kallisti/{kallistiId}/golden-apple"

    expect(@router.routes["chaosMachines.kallisti.eris"].path).
      toEqual "/chaos-machines/{chaosMachinesId}/kallisti/{kallistiId}/eris"

  it "should take options", ->
    @router.draw ->
      @resource "chaosMachines", path: "/fnord"

    expect(@router.routes["chaosMachines"].path).toEqual '/fnord'
    expect(@router.routes["chaosMachines.show"].path).toEqual '/fnord/{id}'
    expect(@router.routes["chaosMachines.edit"].path).toEqual '/fnord/{id}/edit'
{% endhighlight %}

I plan on bundling it into a repo when I get time.
