<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<title>plotly</title>
<script>(function() {
  // If window.HTMLWidgets is already defined, then use it; otherwise create a
  // new object. This allows preceding code to set options that affect the
  // initialization process (though none currently exist).
  window.HTMLWidgets = window.HTMLWidgets || {};

  // See if we're running in a viewer pane. If not, we're in a web browser.
  var viewerMode = window.HTMLWidgets.viewerMode =
      /\bviewer_pane=1\b/.test(window.location);

  // See if we're running in Shiny mode. If not, it's a static document.
  // Note that static widgets can appear in both Shiny and static modes, but
  // obviously, Shiny widgets can only appear in Shiny apps/documents.
  var shinyMode = window.HTMLWidgets.shinyMode =
      typeof(window.Shiny) !== "undefined" && !!window.Shiny.outputBindings;

  // We can't count on jQuery being available, so we implement our own
  // version if necessary.
  function querySelectorAll(scope, selector) {
    if (typeof(jQuery) !== "undefined" && scope instanceof jQuery) {
      return scope.find(selector);
    }
    if (scope.querySelectorAll) {
      return scope.querySelectorAll(selector);
    }
  }

  function asArray(value) {
    if (value === null)
      return [];
    if ($.isArray(value))
      return value;
    return [value];
  }

  // Implement jQuery's extend
  function extend(target /*, ... */) {
    if (arguments.length == 1) {
      return target;
    }
    for (var i = 1; i < arguments.length; i++) {
      var source = arguments[i];
      for (var prop in source) {
        if (source.hasOwnProperty(prop)) {
          target[prop] = source[prop];
        }
      }
    }
    return target;
  }

  // IE8 doesn't support Array.forEach.
  function forEach(values, callback, thisArg) {
    if (values.forEach) {
      values.forEach(callback, thisArg);
    } else {
      for (var i = 0; i < values.length; i++) {
        callback.call(thisArg, values[i], i, values);
      }
    }
  }

  // Replaces the specified method with the return value of funcSource.
  //
  // Note that funcSource should not BE the new method, it should be a function
  // that RETURNS the new method. funcSource receives a single argument that is
  // the overridden method, it can be called from the new method. The overridden
  // method can be called like a regular function, it has the target permanently
  // bound to it so "this" will work correctly.
  function overrideMethod(target, methodName, funcSource) {
    var superFunc = target[methodName] || function() {};
    var superFuncBound = function() {
      return superFunc.apply(target, arguments);
    };
    target[methodName] = funcSource(superFuncBound);
  }

  // Add a method to delegator that, when invoked, calls
  // delegatee.methodName. If there is no such method on
  // the delegatee, but there was one on delegator before
  // delegateMethod was called, then the original version
  // is invoked instead.
  // For example:
  //
  // var a = {
  //   method1: function() { console.log('a1'); }
  //   method2: function() { console.log('a2'); }
  // };
  // var b = {
  //   method1: function() { console.log('b1'); }
  // };
  // delegateMethod(a, b, "method1");
  // delegateMethod(a, b, "method2");
  // a.method1();
  // a.method2();
  //
  // The output would be "b1", "a2".
  function delegateMethod(delegator, delegatee, methodName) {
    var inherited = delegator[methodName];
    delegator[methodName] = function() {
      var target = delegatee;
      var method = delegatee[methodName];

      // The method doesn't exist on the delegatee. Instead,
      // call the method on the delegator, if it exists.
      if (!method) {
        target = delegator;
        method = inherited;
      }

      if (method) {
        return method.apply(target, arguments);
      }
    };
  }

  // Implement a vague facsimilie of jQuery's data method
  function elementData(el, name, value) {
    if (arguments.length == 2) {
      return el["htmlwidget_data_" + name];
    } else if (arguments.length == 3) {
      el["htmlwidget_data_" + name] = value;
      return el;
    } else {
      throw new Error("Wrong number of arguments for elementData: " +
        arguments.length);
    }
  }

  // http://stackoverflow.com/questions/3446170/escape-string-for-use-in-javascript-regex
  function escapeRegExp(str) {
    return str.replace(/[\-\[\]\/\{\}\(\)\*\+\?\.\\\^\$\|]/g, "\\$&");
  }

  function hasClass(el, className) {
    var re = new RegExp("\\b" + escapeRegExp(className) + "\\b");
    return re.test(el.className);
  }

  // elements - array (or array-like object) of HTML elements
  // className - class name to test for
  // include - if true, only return elements with given className;
  //   if false, only return elements *without* given className
  function filterByClass(elements, className, include) {
    var results = [];
    for (var i = 0; i < elements.length; i++) {
      if (hasClass(elements[i], className) == include)
        results.push(elements[i]);
    }
    return results;
  }

  function on(obj, eventName, func) {
    if (obj.addEventListener) {
      obj.addEventListener(eventName, func, false);
    } else if (obj.attachEvent) {
      obj.attachEvent(eventName, func);
    }
  }

  function off(obj, eventName, func) {
    if (obj.removeEventListener)
      obj.removeEventListener(eventName, func, false);
    else if (obj.detachEvent) {
      obj.detachEvent(eventName, func);
    }
  }

  // Translate array of values to top/right/bottom/left, as usual with
  // the "padding" CSS property
  // https://developer.mozilla.org/en-US/docs/Web/CSS/padding
  function unpackPadding(value) {
    if (typeof(value) === "number")
      value = [value];
    if (value.length === 1) {
      return {top: value[0], right: value[0], bottom: value[0], left: value[0]};
    }
    if (value.length === 2) {
      return {top: value[0], right: value[1], bottom: value[0], left: value[1]};
    }
    if (value.length === 3) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[1]};
    }
    if (value.length === 4) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[3]};
    }
  }

  // Convert an unpacked padding object to a CSS value
  function paddingToCss(paddingObj) {
    return paddingObj.top + "px " + paddingObj.right + "px " + paddingObj.bottom + "px " + paddingObj.left + "px";
  }

  // Makes a number suitable for CSS
  function px(x) {
    if (typeof(x) === "number")
      return x + "px";
    else
      return x;
  }

  // Retrieves runtime widget sizing information for an element.
  // The return value is either null, or an object with fill, padding,
  // defaultWidth, defaultHeight fields.
  function sizingPolicy(el) {
    var sizingEl = document.querySelector("script[data-for='" + el.id + "'][type='application/htmlwidget-sizing']");
    if (!sizingEl)
      return null;
    var sp = JSON.parse(sizingEl.textContent || sizingEl.text || "{}");
    if (viewerMode) {
      return sp.viewer;
    } else {
      return sp.browser;
    }
  }

  // @param tasks Array of strings (or falsy value, in which case no-op).
  //   Each element must be a valid JavaScript expression that yields a
  //   function. Or, can be an array of objects with "code" and "data"
  //   properties; in this case, the "code" property should be a string
  //   of JS that's an expr that yields a function, and "data" should be
  //   an object that will be added as an additional argument when that
  //   function is called.
  // @param target The object that will be "this" for each function
  //   execution.
  // @param args Array of arguments to be passed to the functions. (The
  //   same arguments will be passed to all functions.)
  function evalAndRun(tasks, target, args) {
    if (tasks) {
      forEach(tasks, function(task) {
        var theseArgs = args;
        if (typeof(task) === "object") {
          theseArgs = theseArgs.concat([task.data]);
          task = task.code;
        }
        var taskFunc = tryEval(task);
        if (typeof(taskFunc) !== "function") {
          throw new Error("Task must be a function! Source:\n" + task);
        }
        taskFunc.apply(target, theseArgs);
      });
    }
  }

  // Attempt eval() both with and without enclosing in parentheses.
  // Note that enclosing coerces a function declaration into
  // an expression that eval() can parse
  // (otherwise, a SyntaxError is thrown)
  function tryEval(code) {
    var result = null;
    try {
      result = eval(code);
    } catch(error) {
      if (!error instanceof SyntaxError) {
        throw error;
      }
      try {
        result = eval("(" + code + ")");
      } catch(e) {
        if (e instanceof SyntaxError) {
          throw error;
        } else {
          throw e;
        }
      }
    }
    return result;
  }

  function initSizing(el) {
    var sizing = sizingPolicy(el);
    if (!sizing)
      return;

    var cel = document.getElementById("htmlwidget_container");
    if (!cel)
      return;

    if (typeof(sizing.padding) !== "undefined") {
      document.body.style.margin = "0";
      document.body.style.padding = paddingToCss(unpackPadding(sizing.padding));
    }

    if (sizing.fill) {
      document.body.style.overflow = "hidden";
      document.body.style.width = "100%";
      document.body.style.height = "100%";
      document.documentElement.style.width = "100%";
      document.documentElement.style.height = "100%";
      if (cel) {
        cel.style.position = "absolute";
        var pad = unpackPadding(sizing.padding);
        cel.style.top = pad.top + "px";
        cel.style.right = pad.right + "px";
        cel.style.bottom = pad.bottom + "px";
        cel.style.left = pad.left + "px";
        el.style.width = "100%";
        el.style.height = "100%";
      }

      return {
        getWidth: function() { return cel.offsetWidth; },
        getHeight: function() { return cel.offsetHeight; }
      };

    } else {
      el.style.width = px(sizing.width);
      el.style.height = px(sizing.height);

      return {
        getWidth: function() { return el.offsetWidth; },
        getHeight: function() { return el.offsetHeight; }
      };
    }
  }

  // Default implementations for methods
  var defaults = {
    find: function(scope) {
      return querySelectorAll(scope, "." + this.name);
    },
    renderError: function(el, err) {
      var $el = $(el);

      this.clearError(el);

      // Add all these error classes, as Shiny does
      var errClass = "shiny-output-error";
      if (err.type !== null) {
        // use the classes of the error condition as CSS class names
        errClass = errClass + " " + $.map(asArray(err.type), function(type) {
          return errClass + "-" + type;
        }).join(" ");
      }
      errClass = errClass + " htmlwidgets-error";

      // Is el inline or block? If inline or inline-block, just display:none it
      // and add an inline error.
      var display = $el.css("display");
      $el.data("restore-display-mode", display);

      if (display === "inline" || display === "inline-block") {
        $el.hide();
        if (err.message !== "") {
          var errorSpan = $("<span>").addClass(errClass);
          errorSpan.text(err.message);
          $el.after(errorSpan);
        }
      } else if (display === "block") {
        // If block, add an error just after the el, set visibility:none on the
        // el, and position the error to be on top of the el.
        // Mark it with a unique ID and CSS class so we can remove it later.
        $el.css("visibility", "hidden");
        if (err.message !== "") {
          var errorDiv = $("<div>").addClass(errClass).css("position", "absolute")
            .css("top", el.offsetTop)
            .css("left", el.offsetLeft)
            // setting width can push out the page size, forcing otherwise
            // unnecessary scrollbars to appear and making it impossible for
            // the element to shrink; so use max-width instead
            .css("maxWidth", el.offsetWidth)
            .css("height", el.offsetHeight);
          errorDiv.text(err.message);
          $el.after(errorDiv);

          // Really dumb way to keep the size/position of the error in sync with
          // the parent element as the window is resized or whatever.
          var intId = setInterval(function() {
            if (!errorDiv[0].parentElement) {
              clearInterval(intId);
              return;
            }
            errorDiv
              .css("top", el.offsetTop)
              .css("left", el.offsetLeft)
              .css("maxWidth", el.offsetWidth)
              .css("height", el.offsetHeight);
          }, 500);
        }
      }
    },
    clearError: function(el) {
      var $el = $(el);
      var display = $el.data("restore-display-mode");
      $el.data("restore-display-mode", null);

      if (display === "inline" || display === "inline-block") {
        if (display)
          $el.css("display", display);
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      } else if (display === "block"){
        $el.css("visibility", "inherit");
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      }
    },
    sizing: {}
  };

  // Called by widget bindings to register a new type of widget. The definition
  // object can contain the following properties:
  // - name (required) - A string indicating the binding name, which will be
  //   used by default as the CSS classname to look for.
  // - initialize (optional) - A function(el) that will be called once per
  //   widget element; if a value is returned, it will be passed as the third
  //   value to renderValue.
  // - renderValue (required) - A function(el, data, initValue) that will be
  //   called with data. Static contexts will cause this to be called once per
  //   element; Shiny apps will cause this to be called multiple times per
  //   element, as the data changes.
  window.HTMLWidgets.widget = function(definition) {
    if (!definition.name) {
      throw new Error("Widget must have a name");
    }
    if (!definition.type) {
      throw new Error("Widget must have a type");
    }
    // Currently we only support output widgets
    if (definition.type !== "output") {
      throw new Error("Unrecognized widget type '" + definition.type + "'");
    }
    // TODO: Verify that .name is a valid CSS classname

    // Support new-style instance-bound definitions. Old-style class-bound
    // definitions have one widget "object" per widget per type/class of
    // widget; the renderValue and resize methods on such widget objects
    // take el and instance arguments, because the widget object can't
    // store them. New-style instance-bound definitions have one widget
    // object per widget instance; the definition that's passed in doesn't
    // provide renderValue or resize methods at all, just the single method
    //   factory(el, width, height)
    // which returns an object that has renderValue(x) and resize(w, h).
    // This enables a far more natural programming style for the widget
    // author, who can store per-instance state using either OO-style
    // instance fields or functional-style closure variables (I guess this
    // is in contrast to what can only be called C-style pseudo-OO which is
    // what we required before).
    if (definition.factory) {
      definition = createLegacyDefinitionAdapter(definition);
    }

    if (!definition.renderValue) {
      throw new Error("Widget must have a renderValue function");
    }

    // For static rendering (non-Shiny), use a simple widget registration
    // scheme. We also use this scheme for Shiny apps/documents that also
    // contain static widgets.
    window.HTMLWidgets.widgets = window.HTMLWidgets.widgets || [];
    // Merge defaults into the definition; don't mutate the original definition.
    var staticBinding = extend({}, defaults, definition);
    overrideMethod(staticBinding, "find", function(superfunc) {
      return function(scope) {
        var results = superfunc(scope);
        // Filter out Shiny outputs, we only want the static kind
        return filterByClass(results, "html-widget-output", false);
      };
    });
    window.HTMLWidgets.widgets.push(staticBinding);

    if (shinyMode) {
      // Shiny is running. Register the definition with an output binding.
      // The definition itself will not be the output binding, instead
      // we will make an output binding object that delegates to the
      // definition. This is because we foolishly used the same method
      // name (renderValue) for htmlwidgets definition and Shiny bindings
      // but they actually have quite different semantics (the Shiny
      // bindings receive data that includes lots of metadata that it
      // strips off before calling htmlwidgets renderValue). We can't
      // just ignore the difference because in some widgets it's helpful
      // to call this.renderValue() from inside of resize(), and if
      // we're not delegating, then that call will go to the Shiny
      // version instead of the htmlwidgets version.

      // Merge defaults with definition, without mutating either.
      var bindingDef = extend({}, defaults, definition);

      // This object will be our actual Shiny binding.
      var shinyBinding = new Shiny.OutputBinding();

      // With a few exceptions, we'll want to simply use the bindingDef's
      // version of methods if they are available, otherwise fall back to
      // Shiny's defaults. NOTE: If Shiny's output bindings gain additional
      // methods in the future, and we want them to be overrideable by
      // HTMLWidget binding definitions, then we'll need to add them to this
      // list.
      delegateMethod(shinyBinding, bindingDef, "getId");
      delegateMethod(shinyBinding, bindingDef, "onValueChange");
      delegateMethod(shinyBinding, bindingDef, "onValueError");
      delegateMethod(shinyBinding, bindingDef, "renderError");
      delegateMethod(shinyBinding, bindingDef, "clearError");
      delegateMethod(shinyBinding, bindingDef, "showProgress");

      // The find, renderValue, and resize are handled differently, because we
      // want to actually decorate the behavior of the bindingDef methods.

      shinyBinding.find = function(scope) {
        var results = bindingDef.find(scope);

        // Only return elements that are Shiny outputs, not static ones
        var dynamicResults = results.filter(".html-widget-output");

        // It's possible that whatever caused Shiny to think there might be
        // new dynamic outputs, also caused there to be new static outputs.
        // Since there might be lots of different htmlwidgets bindings, we
        // schedule execution for later--no need to staticRender multiple
        // times.
        if (results.length !== dynamicResults.length)
          scheduleStaticRender();

        return dynamicResults;
      };

      // Wrap renderValue to handle initialization, which unfortunately isn't
      // supported natively by Shiny at the time of this writing.

      shinyBinding.renderValue = function(el, data) {
        Shiny.renderDependencies(data.deps);
        // Resolve strings marked as javascript literals to objects
        if (!(data.evals instanceof Array)) data.evals = [data.evals];
        for (var i = 0; data.evals && i < data.evals.length; i++) {
          window.HTMLWidgets.evaluateStringMember(data.x, data.evals[i]);
        }
        if (!bindingDef.renderOnNullValue) {
          if (data.x === null) {
            el.style.visibility = "hidden";
            return;
          } else {
            el.style.visibility = "inherit";
          }
        }
        if (!elementData(el, "initialized")) {
          initSizing(el);

          elementData(el, "initialized", true);
          if (bindingDef.initialize) {
            var result = bindingDef.initialize(el, el.offsetWidth,
              el.offsetHeight);
            elementData(el, "init_result", result);
          }
        }
        bindingDef.renderValue(el, data.x, elementData(el, "init_result"));
        evalAndRun(data.jsHooks.render, elementData(el, "init_result"), [el, data.x]);
      };

      // Only override resize if bindingDef implements it
      if (bindingDef.resize) {
        shinyBinding.resize = function(el, width, height) {
          // Shiny can call resize before initialize/renderValue have been
          // called, which doesn't make sense for widgets.
          if (elementData(el, "initialized")) {
            bindingDef.resize(el, width, height, elementData(el, "init_result"));
          }
        };
      }

      Shiny.outputBindings.register(shinyBinding, bindingDef.name);
    }
  };

  var scheduleStaticRenderTimerId = null;
  function scheduleStaticRender() {
    if (!scheduleStaticRenderTimerId) {
      scheduleStaticRenderTimerId = setTimeout(function() {
        scheduleStaticRenderTimerId = null;
        window.HTMLWidgets.staticRender();
      }, 1);
    }
  }

  // Render static widgets after the document finishes loading
  // Statically render all elements that are of this widget's class
  window.HTMLWidgets.staticRender = function() {
    var bindings = window.HTMLWidgets.widgets || [];
    forEach(bindings, function(binding) {
      var matches = binding.find(document.documentElement);
      forEach(matches, function(el) {
        var sizeObj = initSizing(el, binding);

        if (hasClass(el, "html-widget-static-bound"))
          return;
        el.className = el.className + " html-widget-static-bound";

        var initResult;
        if (binding.initialize) {
          initResult = binding.initialize(el,
            sizeObj ? sizeObj.getWidth() : el.offsetWidth,
            sizeObj ? sizeObj.getHeight() : el.offsetHeight
          );
          elementData(el, "init_result", initResult);
        }

        if (binding.resize) {
          var lastSize = {
            w: sizeObj ? sizeObj.getWidth() : el.offsetWidth,
            h: sizeObj ? sizeObj.getHeight() : el.offsetHeight
          };
          var resizeHandler = function(e) {
            var size = {
              w: sizeObj ? sizeObj.getWidth() : el.offsetWidth,
              h: sizeObj ? sizeObj.getHeight() : el.offsetHeight
            };
            if (size.w === 0 && size.h === 0)
              return;
            if (size.w === lastSize.w && size.h === lastSize.h)
              return;
            lastSize = size;
            binding.resize(el, size.w, size.h, initResult);
          };

          on(window, "resize", resizeHandler);

          // This is needed for cases where we're running in a Shiny
          // app, but the widget itself is not a Shiny output, but
          // rather a simple static widget. One example of this is
          // an rmarkdown document that has runtime:shiny and widget
          // that isn't in a render function. Shiny only knows to
          // call resize handlers for Shiny outputs, not for static
          // widgets, so we do it ourselves.
          if (window.jQuery) {
            window.jQuery(document).on(
              "shown.htmlwidgets shown.bs.tab.htmlwidgets shown.bs.collapse.htmlwidgets",
              resizeHandler
            );
            window.jQuery(document).on(
              "hidden.htmlwidgets hidden.bs.tab.htmlwidgets hidden.bs.collapse.htmlwidgets",
              resizeHandler
            );
          }

          // This is needed for the specific case of ioslides, which
          // flips slides between display:none and display:block.
          // Ideally we would not have to have ioslide-specific code
          // here, but rather have ioslides raise a generic event,
          // but the rmarkdown package just went to CRAN so the
          // window to getting that fixed may be long.
          if (window.addEventListener) {
            // It's OK to limit this to window.addEventListener
            // browsers because ioslides itself only supports
            // such browsers.
            on(document, "slideenter", resizeHandler);
            on(document, "slideleave", resizeHandler);
          }
        }

        var scriptData = document.querySelector("script[data-for='" + el.id + "'][type='application/json']");
        if (scriptData) {
          var data = JSON.parse(scriptData.textContent || scriptData.text);
          // Resolve strings marked as javascript literals to objects
          if (!(data.evals instanceof Array)) data.evals = [data.evals];
          for (var k = 0; data.evals && k < data.evals.length; k++) {
            window.HTMLWidgets.evaluateStringMember(data.x, data.evals[k]);
          }
          binding.renderValue(el, data.x, initResult);
          evalAndRun(data.jsHooks.render, initResult, [el, data.x]);
        }
      });
    });

    invokePostRenderHandlers();
  }


  function has_jQuery3() {
    if (!window.jQuery) {
      return false;
    }
    var $version = window.jQuery.fn.jquery;
    var $major_version = parseInt($version.split(".")[0]);
    return $major_version >= 3;
  }

  /*
  / Shiny 1.4 bumped jQuery from 1.x to 3.x which means jQuery's
  / on-ready handler (i.e., $(fn)) is now asyncronous (i.e., it now
  / really means $(setTimeout(fn)).
  / https://jquery.com/upgrade-guide/3.0/#breaking-change-document-ready-handlers-are-now-asynchronous
  /
  / Since Shiny uses $() to schedule initShiny, shiny>=1.4 calls initShiny
  / one tick later than it did before, which means staticRender() is
  / called renderValue() earlier than (advanced) widget authors might be expecting.
  / https://github.com/rstudio/shiny/issues/2630
  /
  / For a concrete example, leaflet has some methods (e.g., updateBounds)
  / which reference Shiny methods registered in initShiny (e.g., setInputValue).
  / Since leaflet is privy to this life-cycle, it knows to use setTimeout() to
  / delay execution of those methods (until Shiny methods are ready)
  / https://github.com/rstudio/leaflet/blob/18ec981/javascript/src/index.js#L266-L268
  /
  / Ideally widget authors wouldn't need to use this setTimeout() hack that
  / leaflet uses to call Shiny methods on a staticRender(). In the long run,
  / the logic initShiny should be broken up so that method registration happens
  / right away, but binding happens later.
  */
  function maybeStaticRenderLater() {
    if (shinyMode && has_jQuery3()) {
      window.jQuery(window.HTMLWidgets.staticRender);
    } else {
      window.HTMLWidgets.staticRender();
    }
  }

  if (document.addEventListener) {
    document.addEventListener("DOMContentLoaded", function() {
      document.removeEventListener("DOMContentLoaded", arguments.callee, false);
      maybeStaticRenderLater();
    }, false);
  } else if (document.attachEvent) {
    document.attachEvent("onreadystatechange", function() {
      if (document.readyState === "complete") {
        document.detachEvent("onreadystatechange", arguments.callee);
        maybeStaticRenderLater();
      }
    });
  }


  window.HTMLWidgets.getAttachmentUrl = function(depname, key) {
    // If no key, default to the first item
    if (typeof(key) === "undefined")
      key = 1;

    var link = document.getElementById(depname + "-" + key + "-attachment");
    if (!link) {
      throw new Error("Attachment " + depname + "/" + key + " not found in document");
    }
    return link.getAttribute("href");
  };

  window.HTMLWidgets.dataframeToD3 = function(df) {
    var names = [];
    var length;
    for (var name in df) {
        if (df.hasOwnProperty(name))
            names.push(name);
        if (typeof(df[name]) !== "object" || typeof(df[name].length) === "undefined") {
            throw new Error("All fields must be arrays");
        } else if (typeof(length) !== "undefined" && length !== df[name].length) {
            throw new Error("All fields must be arrays of the same length");
        }
        length = df[name].length;
    }
    var results = [];
    var item;
    for (var row = 0; row < length; row++) {
        item = {};
        for (var col = 0; col < names.length; col++) {
            item[names[col]] = df[names[col]][row];
        }
        results.push(item);
    }
    return results;
  };

  window.HTMLWidgets.transposeArray2D = function(array) {
      if (array.length === 0) return array;
      var newArray = array[0].map(function(col, i) {
          return array.map(function(row) {
              return row[i]
          })
      });
      return newArray;
  };
  // Split value at splitChar, but allow splitChar to be escaped
  // using escapeChar. Any other characters escaped by escapeChar
  // will be included as usual (including escapeChar itself).
  function splitWithEscape(value, splitChar, escapeChar) {
    var results = [];
    var escapeMode = false;
    var currentResult = "";
    for (var pos = 0; pos < value.length; pos++) {
      if (!escapeMode) {
        if (value[pos] === splitChar) {
          results.push(currentResult);
          currentResult = "";
        } else if (value[pos] === escapeChar) {
          escapeMode = true;
        } else {
          currentResult += value[pos];
        }
      } else {
        currentResult += value[pos];
        escapeMode = false;
      }
    }
    if (currentResult !== "") {
      results.push(currentResult);
    }
    return results;
  }
  // Function authored by Yihui/JJ Allaire
  window.HTMLWidgets.evaluateStringMember = function(o, member) {
    var parts = splitWithEscape(member, '.', '\\');
    for (var i = 0, l = parts.length; i < l; i++) {
      var part = parts[i];
      // part may be a character or 'numeric' member name
      if (o !== null && typeof o === "object" && part in o) {
        if (i == (l - 1)) { // if we are at the end of the line then evalulate
          if (typeof o[part] === "string")
            o[part] = tryEval(o[part]);
        } else { // otherwise continue to next embedded object
          o = o[part];
        }
      }
    }
  };

  // Retrieve the HTMLWidget instance (i.e. the return value of an
  // HTMLWidget binding's initialize() or factory() function)
  // associated with an element, or null if none.
  window.HTMLWidgets.getInstance = function(el) {
    return elementData(el, "init_result");
  };

  // Finds the first element in the scope that matches the selector,
  // and returns the HTMLWidget instance (i.e. the return value of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with that element, if any. If no element matches the
  // selector, or the first matching element has no HTMLWidget
  // instance associated with it, then null is returned.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.find = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var el = scope.querySelector(selector);
    if (el === null) {
      return null;
    } else {
      return window.HTMLWidgets.getInstance(el);
    }
  };

  // Finds all elements in the scope that match the selector, and
  // returns the HTMLWidget instances (i.e. the return values of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with the elements, in an array. If elements that
  // match the selector don't have an associated HTMLWidget
  // instance, the returned array will contain nulls.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.findAll = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var nodes = scope.querySelectorAll(selector);
    var results = [];
    for (var i = 0; i < nodes.length; i++) {
      results.push(window.HTMLWidgets.getInstance(nodes[i]));
    }
    return results;
  };

  var postRenderHandlers = [];
  function invokePostRenderHandlers() {
    while (postRenderHandlers.length) {
      var handler = postRenderHandlers.shift();
      if (handler) {
        handler();
      }
    }
  }

  // Register the given callback function to be invoked after the
  // next time static widgets are rendered.
  window.HTMLWidgets.addPostRenderHandler = function(callback) {
    postRenderHandlers.push(callback);
  };

  // Takes a new-style instance-bound definition, and returns an
  // old-style class-bound definition. This saves us from having
  // to rewrite all the logic in this file to accomodate both
  // types of definitions.
  function createLegacyDefinitionAdapter(defn) {
    var result = {
      name: defn.name,
      type: defn.type,
      initialize: function(el, width, height) {
        return defn.factory(el, width, height);
      },
      renderValue: function(el, x, instance) {
        return instance.renderValue(x);
      },
      resize: function(el, width, height, instance) {
        return instance.resize(width, height);
      }
    };

    if (defn.find)
      result.find = defn.find;
    if (defn.renderError)
      result.renderError = defn.renderError;
    if (defn.clearError)
      result.clearError = defn.clearError;

    return result;
  }
})();

</script>
<script>
HTMLWidgets.widget({
  name: "plotly",
  type: "output",

  initialize: function(el, width, height) {
    return {};
  },

  resize: function(el, width, height, instance) {
    if (instance.autosize) {
      var width = instance.width || width;
      var height = instance.height || height;
      Plotly.relayout(el.id, {width: width, height: height});
    }
  },  
  
  renderValue: function(el, x, instance) {
    
    // Plotly.relayout() mutates the plot input object, so make sure to 
    // keep a reference to the user-supplied width/height *before*
    // we call Plotly.plot();
    var lay = x.layout || {};
    instance.width = lay.width;
    instance.height = lay.height;
    instance.autosize = lay.autosize || true;
    
    /* 
    / 'inform the world' about highlighting options this is so other
    / crosstalk libraries have a chance to respond to special settings 
    / such as persistent selection. 
    / AFAIK, leaflet is the only library with such intergration
    / https://github.com/rstudio/leaflet/pull/346/files#diff-ad0c2d51ce5fdf8c90c7395b102f4265R154
    */
    var ctConfig = crosstalk.var('plotlyCrosstalkOpts').set(x.highlight);
      
    if (typeof(window) !== "undefined") {
      // make sure plots don't get created outside the network (for on-prem)
      window.PLOTLYENV = window.PLOTLYENV || {};
      window.PLOTLYENV.BASE_URL = x.base_url;
      
      // Enable persistent selection when shift key is down
      // https://stackoverflow.com/questions/1828613/check-if-a-key-is-down
      var persistOnShift = function(e) {
        if (!e) window.event;
        if (e.shiftKey) { 
          x.highlight.persistent = true; 
          x.highlight.persistentShift = true;
        } else {
          x.highlight.persistent = false; 
          x.highlight.persistentShift = false;
        }
      };
      
      // Only relevant if we haven't forced persistent mode at command line
      if (!x.highlight.persistent) {
        window.onmousemove = persistOnShift;
      }
    }

    var graphDiv = document.getElementById(el.id);
    
    // TODO: move the control panel injection strategy inside here...
    HTMLWidgets.addPostRenderHandler(function() {
      
      // lower the z-index of the modebar to prevent it from highjacking hover
      // (TODO: do this via CSS?)
      // https://github.com/ropensci/plotly/issues/956
      // https://www.w3schools.com/jsref/prop_style_zindex.asp
      var modebars = document.querySelectorAll(".js-plotly-plot .plotly .modebar");
      for (var i = 0; i < modebars.length; i++) {
        modebars[i].style.zIndex = 1;
      }
    });
      
      // inject a "control panel" holding selectize/dynamic color widget(s)
    if (x.selectize || x.highlight.dynamic && !instance.plotly) {
      var flex = document.createElement("div");
      flex.class = "plotly-crosstalk-control-panel";
      flex.style = "display: flex; flex-wrap: wrap";
      
      // inject the colourpicker HTML container into the flexbox
      if (x.highlight.dynamic) {
        var pickerDiv = document.createElement("div");
        
        var pickerInput = document.createElement("input");
        pickerInput.id = el.id + "-colourpicker";
        pickerInput.placeholder = "asdasd";
        
        var pickerLabel = document.createElement("label");
        pickerLabel.for = pickerInput.id;
        pickerLabel.innerHTML = "Brush color&nbsp;&nbsp;";
        
        pickerDiv.appendChild(pickerLabel);
        pickerDiv.appendChild(pickerInput);
        flex.appendChild(pickerDiv);
      }
      
      // inject selectize HTML containers (one for every crosstalk group)
      if (x.selectize) {
        var ids = Object.keys(x.selectize);
        
        for (var i = 0; i < ids.length; i++) {
          var container = document.createElement("div");
          container.id = ids[i];
          container.style = "width: 80%; height: 10%";
          container.class = "form-group crosstalk-input-plotly-highlight";
          
          var label = document.createElement("label");
          label.for = ids[i];
          label.innerHTML = x.selectize[ids[i]].group;
          label.class = "control-label";
          
          var selectDiv = document.createElement("div");
          var select = document.createElement("select");
          select.multiple = true;
          
          selectDiv.appendChild(select);
          container.appendChild(label);
          container.appendChild(selectDiv);
          flex.appendChild(container);
        }
      }
      
      // finally, insert the flexbox inside the htmlwidget container,
      // but before the plotly graph div
      graphDiv.parentElement.insertBefore(flex, graphDiv);
      
      if (x.highlight.dynamic) {
        var picker = $("#" + pickerInput.id);
        var colors = x.highlight.color || [];
        // TODO: let users specify options?
        var opts = {
          value: colors[0],
          showColour: "both",
          palette: "limited",
          allowedCols: colors.join(" "),
          width: "20%",
          height: "10%"
        };
        picker.colourpicker({changeDelay: 0});
        picker.colourpicker("settings", opts);
        picker.colourpicker("value", opts.value);
        // inform crosstalk about a change in the current selection colour
        var grps = x.highlight.ctGroups || [];
        for (var i = 0; i < grps.length; i++) {
          crosstalk.group(grps[i]).var('plotlySelectionColour')
            .set(picker.colourpicker('value'));
        }
        picker.on("change", function() {
          for (var i = 0; i < grps.length; i++) {
            crosstalk.group(grps[i]).var('plotlySelectionColour')
              .set(picker.colourpicker('value'));
          }
        });
      }
    }
    
    // if no plot exists yet, create one with a particular configuration
    if (!instance.plotly) {
      
      var plot = Plotly.plot(graphDiv, x);
      instance.plotly = true;
      
    } else {
      
      // this is essentially equivalent to Plotly.newPlot(), but avoids creating 
      // a new webgl context
      // https://github.com/plotly/plotly.js/blob/2b24f9def901831e61282076cf3f835598d56f0e/src/plot_api/plot_api.js#L531-L532

      // TODO: restore crosstalk selections?
      Plotly.purge(graphDiv);
      // TODO: why is this necessary to get crosstalk working?
      graphDiv.data = undefined;
      graphDiv.layout = undefined;
      var plot = Plotly.plot(graphDiv, x);
    }
    
    // Trigger plotly.js calls defined via `plotlyProxy()`
    plot.then(function() {
      if (HTMLWidgets.shinyMode) {
        Shiny.addCustomMessageHandler("plotly-calls", function(msg) {
          var gd = document.getElementById(msg.id);
          if (!gd) {
            throw new Error("Couldn't find plotly graph with id: " + msg.id);
          }
          // This isn't an official plotly.js method, but it's the only current way to 
          // change just the configuration of a plot 
          // https://community.plot.ly/t/update-config-function/9057
          if (msg.method == "reconfig") {
            Plotly.react(gd, gd.data, gd.layout, msg.args);
            return;
          }
          if (!Plotly[msg.method]) {
            throw new Error("Unknown method " + msg.method);
          }
          var args = [gd].concat(msg.args);
          Plotly[msg.method].apply(null, args);
        });
      }
      
      // plotly's mapbox API doesn't currently support setting bounding boxes
      // https://www.mapbox.com/mapbox-gl-js/example/fitbounds/
      // so we do this manually...
      // TODO: make sure this triggers on a redraw and relayout as well as on initial draw
      var mapboxIDs = graphDiv._fullLayout._subplots.mapbox || [];
      for (var i = 0; i < mapboxIDs.length; i++) {
        var id = mapboxIDs[i];
        var mapOpts = x.layout[id] || {};
        var args = mapOpts._fitBounds || {};
        if (!args) {
          continue;
        }
        var mapObj = graphDiv._fullLayout[id]._subplot.map;
        mapObj.fitBounds(args.bounds, args.options);
      }
      
    });
    
    // Attach attributes (e.g., "key", "z") to plotly event data
    function eventDataWithKey(eventData) {
      if (eventData === undefined || !eventData.hasOwnProperty("points")) {
        return null;
      }
      return eventData.points.map(function(pt) {
        var obj = {
          curveNumber: pt.curveNumber, 
          pointNumber: pt.pointNumber, 
          x: pt.x,
          y: pt.y
        };
        
        // If 'z' is reported with the event data, then use it!
        if (pt.hasOwnProperty("z")) {
          obj.z = pt.z;
        }
        
        if (pt.hasOwnProperty("customdata")) {
          obj.customdata = pt.customdata;
        }
        
        /* 
          TL;DR: (I think) we have to select the graph div (again) to attach keys...
          
          Why? Remember that crosstalk will dynamically add/delete traces 
          (see traceManager.prototype.updateSelection() below)
          For this reason, we can't simply grab keys from x.data (like we did previously)
          Moreover, we can't use _fullData, since that doesn't include 
          unofficial attributes. It's true that click/hover events fire with 
          pt.data, but drag events don't...
        */
        var gd = document.getElementById(el.id);
        var trace = gd.data[pt.curveNumber];
        
        if (!trace._isSimpleKey) {
          var attrsToAttach = ["key"];
        } else {
          // simple keys fire the whole key
          obj.key = trace.key;
          var attrsToAttach = [];
        }
        
        for (var i = 0; i < attrsToAttach.length; i++) {
          var attr = trace[attrsToAttach[i]];
          if (Array.isArray(attr)) {
            if (typeof pt.pointNumber === "number") {
              obj[attrsToAttach[i]] = attr[pt.pointNumber];
            } else if (Array.isArray(pt.pointNumber)) {
              obj[attrsToAttach[i]] = attr[pt.pointNumber[0]][pt.pointNumber[1]];
            } else if (Array.isArray(pt.pointNumbers)) {
              obj[attrsToAttach[i]] = pt.pointNumbers.map(function(idx) { return attr[idx]; });
            }
          }
        }
        return obj;
      });
    }
    
    
    var legendEventData = function(d) {
      // if legendgroup is not relevant just return the trace
      var trace = d.data[d.curveNumber];
      if (!trace.legendgroup) return trace;
      
      // if legendgroup was specified, return all traces that match the group
      var legendgrps = d.data.map(function(trace){ return trace.legendgroup; });
      var traces = [];
      for (i = 0; i < legendgrps.length; i++) {
        if (legendgrps[i] == trace.legendgroup) {
          traces.push(d.data[i]);
        }
      }
      
      return traces;
    };

    
    // send user input event data to shiny
    if (HTMLWidgets.shinyMode && Shiny.setInputValue) {
      
      // Some events clear other input values
      // TODO: always register these?
      var eventClearMap = {
        plotly_deselect: ["plotly_selected", "plotly_selecting", "plotly_brushed", "plotly_brushing", "plotly_click"],
        plotly_unhover: ["plotly_hover"],
        plotly_doubleclick: ["plotly_click"]
      };
    
      Object.keys(eventClearMap).map(function(evt) {
        graphDiv.on(evt, function() {
          var inputsToClear = eventClearMap[evt];
          inputsToClear.map(function(input) {
            Shiny.setInputValue(input + "-" + x.source, null, {priority: "event"});
          });
        });
      });
      
      var eventDataFunctionMap = {
        plotly_click: eventDataWithKey,
        plotly_sunburstclick: eventDataWithKey,
        plotly_hover: eventDataWithKey,
        plotly_unhover: eventDataWithKey,
        // If 'plotly_selected' has already been fired, and you click
        // on the plot afterwards, this event fires `undefined`?!?
        // That might be considered a plotly.js bug, but it doesn't make 
        // sense for this input change to occur if `d` is falsy because,
        // even in the empty selection case, `d` is truthy (an object),
        // and the 'plotly_deselect' event will reset this input
        plotly_selected: function(d) { if (d) { return eventDataWithKey(d); } },
        plotly_selecting: function(d) { if (d) { return eventDataWithKey(d); } },
        plotly_brushed: function(d) {
          if (d) { return d.range ? d.range : d.lassoPoints; }
        },
        plotly_brushing: function(d) {
          if (d) { return d.range ? d.range : d.lassoPoints; }
        },
        plotly_legendclick: legendEventData,
        plotly_legenddoubleclick: legendEventData,
        plotly_clickannotation: function(d) { return d.fullAnnotation }
      };
      
      var registerShinyValue = function(event) {
        var eventDataPreProcessor = eventDataFunctionMap[event] || function(d) { return d ? d : el.id };
        // some events are unique to the R package
        var plotlyJSevent = (event == "plotly_brushed") ? "plotly_selected" : (event == "plotly_brushing") ? "plotly_selecting" : event;
        // register the event
        graphDiv.on(plotlyJSevent, function(d) {
          Shiny.setInputValue(
            event + "-" + x.source,
            JSON.stringify(eventDataPreProcessor(d)),
            {priority: "event"}
          );
        });
      }
    
      var shinyEvents = x.shinyEvents || [];
      shinyEvents.map(registerShinyValue);
    }
    
    // Given an array of {curveNumber: x, pointNumber: y} objects,
    // return a hash of {
    //   set1: {value: [key1, key2, ...], _isSimpleKey: false}, 
    //   set2: {value: [key3, key4, ...], _isSimpleKey: false}
    // }
    function pointsToKeys(points) {
      var keysBySet = {};
      for (var i = 0; i < points.length; i++) {
        
        var trace = graphDiv.data[points[i].curveNumber];
        if (!trace.key || !trace.set) {
          continue;
        }
        
        // set defaults for this keySet
        // note that we don't track the nested property (yet) since we always 
        // emit the union -- http://cpsievert.github.io/talks/20161212b/#21
        keysBySet[trace.set] = keysBySet[trace.set] || {
          value: [],
          _isSimpleKey: trace._isSimpleKey
        };
        
        // Use pointNumber by default, but aggregated traces should emit pointNumbers
        var ptNum = points[i].pointNumber;
        var hasPtNum = typeof ptNum === "number";
        var ptNum = hasPtNum ? ptNum : points[i].pointNumbers;
        
        // selecting a point of a "simple" trace means: select the 
        // entire key attached to this trace, which is useful for,
        // say clicking on a fitted line to select corresponding observations 
        var key = trace._isSimpleKey ? trace.key : Array.isArray(ptNum) ? ptNum.map(function(idx) { return trace.key[idx]; }) : trace.key[ptNum];
        // http://stackoverflow.com/questions/10865025/merge-flatten-an-array-of-arrays-in-javascript
        var keyFlat = trace._isNestedKey ? [].concat.apply([], key) : key;
        
        // TODO: better to only add new values?
        keysBySet[trace.set].value = keysBySet[trace.set].value.concat(keyFlat);
      }
      
      return keysBySet;
    }
    
    
    x.highlight.color = x.highlight.color || [];
    // make sure highlight color is an array
    if (!Array.isArray(x.highlight.color)) {
      x.highlight.color = [x.highlight.color];
    }

    var traceManager = new TraceManager(graphDiv, x.highlight);

    // Gather all *unique* sets.
    var allSets = [];
    for (var curveIdx = 0; curveIdx < x.data.length; curveIdx++) {
      var newSet = x.data[curveIdx].set;
      if (newSet) {
        if (allSets.indexOf(newSet) === -1) {
          allSets.push(newSet);
        }
      }
    }

    // register event listeners for all sets
    for (var i = 0; i < allSets.length; i++) {
      
      var set = allSets[i];
      var selection = new crosstalk.SelectionHandle(set);
      var filter = new crosstalk.FilterHandle(set);
      
      var filterChange = function(e) {
        removeBrush(el);
        traceManager.updateFilter(set, e.value);
      };
      filter.on("change", filterChange);
      
      
      var selectionChange = function(e) {
        
        // Workaround for 'plotly_selected' now firing previously selected
        // points (in addition to new ones) when holding shift key. In our case,
        // we just want the new keys 
        if (x.highlight.on === "plotly_selected" && x.highlight.persistentShift) {
          // https://stackoverflow.com/questions/1187518/how-to-get-the-difference-between-two-arrays-in-javascript
          Array.prototype.diff = function(a) {
              return this.filter(function(i) {return a.indexOf(i) < 0;});
          };
          e.value = e.value.diff(e.oldValue);
        }
        
        // array of "event objects" tracking the selection history
        // this is used to avoid adding redundant selections
        var selectionHistory = crosstalk.var("plotlySelectionHistory").get() || [];
        
        // Construct an event object "defining" the current event. 
        var event = {
          receiverID: traceManager.gd.id,
          plotlySelectionColour: crosstalk.group(set).var("plotlySelectionColour").get()
        };
        event[set] = e.value;
        // TODO: is there a smarter way to check object equality?
        if (selectionHistory.length > 0) {
          var ev = JSON.stringify(event);
          for (var i = 0; i < selectionHistory.length; i++) {
            var sel = JSON.stringify(selectionHistory[i]);
            if (sel == ev) {
              return;
            }
          }
        }
        
        // accumulate history for persistent selection
        if (!x.highlight.persistent) {
          selectionHistory = [event];
        } else {
          selectionHistory.push(event);
        }
        crosstalk.var("plotlySelectionHistory").set(selectionHistory);
        
        // do the actual updating of traces, frames, and the selectize widget
        traceManager.updateSelection(set, e.value);
        // https://github.com/selectize/selectize.js/blob/master/docs/api.md#methods_items
        if (x.selectize) {
          if (!x.highlight.persistent || e.value === null) {
            selectize.clear(true);
          }
          selectize.addItems(e.value, true);
          selectize.close();
        }
      }
      selection.on("change", selectionChange);
      
      // Set a crosstalk variable selection value, triggering an update
      var turnOn = function(e) {
        if (e) {
          var selectedKeys = pointsToKeys(e.points);
          // Keys are group names, values are array of selected keys from group.
          for (var set in selectedKeys) {
            if (selectedKeys.hasOwnProperty(set)) {
              selection.set(selectedKeys[set].value, {sender: el});
            }
          }
        }
      };
      if (x.highlight.debounce > 0) {
        turnOn = debounce(turnOn, x.highlight.debounce);
      }
      graphDiv.on(x.highlight.on, turnOn);
      
      graphDiv.on(x.highlight.off, function turnOff(e) {
        // remove any visual clues
        removeBrush(el);
        // remove any selection history
        crosstalk.var("plotlySelectionHistory").set(null);
        // trigger the actual removal of selection traces
        selection.set(null, {sender: el});
      });
          
      // register a callback for selectize so that there is bi-directional
      // communication between the widget and direct manipulation events
      if (x.selectize) {
        var selectizeID = Object.keys(x.selectize)[i];
        var items = x.selectize[selectizeID].items;
        var first = [{value: "", label: "(All)"}];
        var opts = {
          options: first.concat(items),
          searchField: "label",
          valueField: "value",
          labelField: "label",
          maxItems: 50
        };
        var select = $("#" + selectizeID).find("select")[0];
        var selectize = $(select).selectize(opts)[0].selectize;
        // NOTE: this callback is triggered when *directly* altering 
        // dropdown items
        selectize.on("change", function() {
          var currentItems = traceManager.groupSelections[set] || [];
          if (!x.highlight.persistent) {
            removeBrush(el);
            for (var i = 0; i < currentItems.length; i++) {
              selectize.removeItem(currentItems[i], true);
            }
          }
          var newItems = selectize.items.filter(function(idx) { 
            return currentItems.indexOf(idx) < 0;
          });
          if (newItems.length > 0) {
            traceManager.updateSelection(set, newItems);
          } else {
            // Item has been removed...
            // TODO: this logic won't work for dynamically changing palette 
            traceManager.updateSelection(set, null);
            traceManager.updateSelection(set, selectize.items);
          }
        });
      }
    } // end of selectionChange
    
  } // end of renderValue
}); // end of widget definition

/**
 * @param graphDiv The Plotly graph div
 * @param highlight An object with options for updating selection(s)
 */
function TraceManager(graphDiv, highlight) {
  // The Plotly graph div
  this.gd = graphDiv;

  // Preserve the original data.
  // TODO: try using Lib.extendFlat() as done in  
  // https://github.com/plotly/plotly.js/pull/1136 
  this.origData = JSON.parse(JSON.stringify(graphDiv.data));
  
  // avoid doing this over and over
  this.origOpacity = [];
  for (var i = 0; i < this.origData.length; i++) {
    this.origOpacity[i] = this.origData[i].opacity === 0 ? 0 : (this.origData[i].opacity || 1);
  }

  // key: group name, value: null or array of keys representing the
  // most recently received selection for that group.
  this.groupSelections = {};
  
  // selection parameters (e.g., transient versus persistent selection)
  this.highlight = highlight;
}

TraceManager.prototype.close = function() {
  // TODO: Unhook all event handlers
};

TraceManager.prototype.updateFilter = function(group, keys) {

  if (typeof(keys) === "undefined" || keys === null) {
    
    this.gd.data = JSON.parse(JSON.stringify(this.origData));
    
  } else {
  
    var traces = [];
    for (var i = 0; i < this.origData.length; i++) {
      var trace = this.origData[i];
      if (!trace.key || trace.set !== group) {
        continue;
      }
      var matchFunc = getMatchFunc(trace);
      var matches = matchFunc(trace.key, keys);
      
      if (matches.length > 0) {
        if (!trace._isSimpleKey) {
          // subsetArrayAttrs doesn't mutate trace (it makes a modified clone)
          trace = subsetArrayAttrs(trace, matches);
        }
        traces.push(trace);
      }
    }
  }
  
  this.gd.data = traces;
  Plotly.redraw(this.gd);
  
  // NOTE: we purposely do _not_ restore selection(s), since on filter,
  // axis likely will update, changing the pixel -> data mapping, leading 
  // to a likely mismatch in the brush outline and highlighted marks
  
};

TraceManager.prototype.updateSelection = function(group, keys) {
  
  if (keys !== null && !Array.isArray(keys)) {
    throw new Error("Invalid keys argument; null or array expected");
  }
  
  // if selection has been cleared, or if this is transient
  // selection, delete the "selection traces"
  var nNewTraces = this.gd.data.length - this.origData.length;
  if (keys === null || !this.highlight.persistent && nNewTraces > 0) {
    var tracesToRemove = [];
    for (var i = 0; i < this.gd.data.length; i++) {
      if (this.gd.data[i]._isCrosstalkTrace) tracesToRemove.push(i);
    }
    Plotly.deleteTraces(this.gd, tracesToRemove);
    this.groupSelections[group] = keys;
  } else {
    // add to the groupSelection, rather than overwriting it
    // TODO: can this be removed?
    this.groupSelections[group] = this.groupSelections[group] || [];
    for (var i = 0; i < keys.length; i++) {
      var k = keys[i];
      if (this.groupSelections[group].indexOf(k) < 0) {
        this.groupSelections[group].push(k);
      }
    }
  }
  
  if (keys === null) {
    
    Plotly.restyle(this.gd, {"opacity": this.origOpacity});
    
  } else if (keys.length >= 1) {
    
    // placeholder for new "selection traces"
    var traces = [];
    // this variable is set in R/highlight.R
    var selectionColour = crosstalk.group(group).var("plotlySelectionColour").get() || 
      this.highlight.color[0];

    for (var i = 0; i < this.origData.length; i++) {
      // TODO: try using Lib.extendFlat() as done in  
      // https://github.com/plotly/plotly.js/pull/1136 
      var trace = JSON.parse(JSON.stringify(this.gd.data[i]));
      if (!trace.key || trace.set !== group) {
        continue;
      }
      // Get sorted array of matching indices in trace.key
      var matchFunc = getMatchFunc(trace);
      var matches = matchFunc(trace.key, keys);
      
      if (matches.length > 0) {
        // If this is a "simple" key, that means select the entire trace
        if (!trace._isSimpleKey) {
          trace = subsetArrayAttrs(trace, matches);
        }
        // reach into the full trace object so we can properly reflect the 
        // selection attributes in every view
        var d = this.gd._fullData[i];
        
        /* 
        / Recursively inherit selection attributes from various sources, 
        / in order of preference:
        /  (1) official plotly.js selected attribute
        /  (2) highlight(selected = attrs_selected(...))
        */
        // TODO: it would be neat to have a dropdown to dynamically specify these!
        $.extend(true, trace, this.highlight.selected);
        
        // if it is defined, override color with the "dynamic brush color""
        if (d.marker) {
          trace.marker = trace.marker || {};
          trace.marker.color =  selectionColour || trace.marker.color || d.marker.color;
        }
        if (d.line) {
          trace.line = trace.line || {};
          trace.line.color =  selectionColour || trace.line.color || d.line.color;
        }
        if (d.textfont) {
          trace.textfont = trace.textfont || {};
          trace.textfont.color =  selectionColour || trace.textfont.color || d.textfont.color;
        }
        if (d.fillcolor) {
          // TODO: should selectionColour inherit alpha from the existing fillcolor?
          trace.fillcolor = selectionColour || trace.fillcolor || d.fillcolor;
        }
        // attach a sensible name/legendgroup
        trace.name = trace.name || keys.join("<br />");
        trace.legendgroup = trace.legendgroup || keys.join("<br />");
        
        // keep track of mapping between this new trace and the trace it targets
        // (necessary for updating frames to reflect the selection traces)
        trace._originalIndex = i;
        trace._newIndex = this.gd._fullData.length + traces.length;
        trace._isCrosstalkTrace = true;
        traces.push(trace);
      }
    }
    
    if (traces.length > 0) {
      
      Plotly.addTraces(this.gd, traces).then(function(gd) {
        // incrementally add selection traces to frames
        // (this is heavily inspired by Plotly.Plots.modifyFrames() 
        // in src/plots/plots.js)
        var _hash = gd._transitionData._frameHash;
        var _frames = gd._transitionData._frames || [];
        
        for (var i = 0; i < _frames.length; i++) {
          
          // add to _frames[i].traces *if* this frame references selected trace(s)
          var newIndices = [];
          for (var j = 0; j < traces.length; j++) {
            var tr = traces[j];
            if (_frames[i].traces.indexOf(tr._originalIndex) > -1) {
              newIndices.push(tr._newIndex);
              _frames[i].traces.push(tr._newIndex);
            }
          }
          
          // nothing to do...
          if (newIndices.length === 0) {
            continue;
          }
          
          var ctr = 0;
          var nFrameTraces = _frames[i].data.length;
          
          for (var j = 0; j < nFrameTraces; j++) {
            var frameTrace = _frames[i].data[j];
            if (!frameTrace.key || frameTrace.set !== group) {
              continue;
            }
            
            var matchFunc = getMatchFunc(frameTrace);
            var matches = matchFunc(frameTrace.key, keys);
            
            if (matches.length > 0) {
              if (!trace._isSimpleKey) {
                frameTrace = subsetArrayAttrs(frameTrace, matches);
              }
              var d = gd._fullData[newIndices[ctr]];
              if (d.marker) {
                frameTrace.marker = d.marker;
              }
              if (d.line) {
                frameTrace.line = d.line;
              }
              if (d.textfont) {
                frameTrace.textfont = d.textfont;
              }
              ctr = ctr + 1;
              _frames[i].data.push(frameTrace);
            }
          }
          
          // update gd._transitionData._frameHash
          _hash[_frames[i].name] = _frames[i];
        }
      
      });
      
      // dim traces that have a set matching the set of selection sets
      var tracesToDim = [],
          opacities = [],
          sets = Object.keys(this.groupSelections),
          n = this.origData.length;
          
      for (var i = 0; i < n; i++) {
        var opacity = this.origOpacity[i] || 1;
        // have we already dimmed this trace? Or is this even worth doing?
        if (opacity !== this.gd._fullData[i].opacity || this.highlight.opacityDim === 1) {
          continue;
        }
        // is this set an element of the set of selection sets?
        var matches = findMatches(sets, [this.gd.data[i].set]);
        if (matches.length) {
          tracesToDim.push(i);
          opacities.push(opacity * this.highlight.opacityDim);
        }
      }
      
      if (tracesToDim.length > 0) {
        Plotly.restyle(this.gd, {"opacity": opacities}, tracesToDim);
        // turn off the selected/unselected API
        Plotly.restyle(this.gd, {"selectedpoints": null});
      }
      
    }
    
  }
};

/* 
Note: in all of these match functions, we assume needleSet (i.e. the selected keys)
is a 1D (or flat) array. The real difference is the meaning of haystack.
findMatches() does the usual thing you'd expect for 
linked brushing on a scatterplot matrix. findSimpleMatches() returns a match iff 
haystack is a subset of the needleSet. findNestedMatches() returns 
*/

function getMatchFunc(trace) {
  return (trace._isNestedKey) ? findNestedMatches : 
    (trace._isSimpleKey) ? findSimpleMatches : findMatches;
}

// find matches for "flat" keys
function findMatches(haystack, needleSet) {
  var matches = [];
  haystack.forEach(function(obj, i) {
    if (obj === null || needleSet.indexOf(obj) >= 0) {
      matches.push(i);
    }
  });
  return matches;
}

// find matches for "simple" keys
function findSimpleMatches(haystack, needleSet) {
  var match = haystack.every(function(val) {
    return val === null || needleSet.indexOf(val) >= 0;
  });
  // yes, this doesn't make much sense other than conforming 
  // to the output type of the other match functions
  return (match) ? [0] : []
}

// find matches for a "nested" haystack (2D arrays)
function findNestedMatches(haystack, needleSet) {
  var matches = [];
  for (var i = 0; i < haystack.length; i++) {
    var hay = haystack[i];
    var match = hay.every(function(val) { 
      return val === null || needleSet.indexOf(val) >= 0; 
    });
    if (match) {
      matches.push(i);
    }
  }
  return matches;
}

function isPlainObject(obj) {
  return (
    Object.prototype.toString.call(obj) === '[object Object]' &&
    Object.getPrototypeOf(obj) === Object.prototype
  );
}

function subsetArrayAttrs(obj, indices) {
  var newObj = {};
  Object.keys(obj).forEach(function(k) {
    var val = obj[k];

    if (k.charAt(0) === "_") {
      newObj[k] = val;
    } else if (k === "transforms" && Array.isArray(val)) {
      newObj[k] = val.map(function(transform) {
        return subsetArrayAttrs(transform, indices);
      });
    } else if (k === "colorscale" && Array.isArray(val)) {
      newObj[k] = val;
    } else if (isPlainObject(val)) {
      newObj[k] = subsetArrayAttrs(val, indices);
    } else if (Array.isArray(val)) {
      newObj[k] = subsetArray(val, indices);
    } else {
      newObj[k] = val;
    }
  });
  return newObj;
}

function subsetArray(arr, indices) {
  var result = [];
  for (var i = 0; i < indices.length; i++) {
    result.push(arr[indices[i]]);
  }
  return result;
}

// Convenience function for removing plotly's brush 
function removeBrush(el) {
  var outlines = el.querySelectorAll(".select-outline");
  for (var i = 0; i < outlines.length; i++) {
    outlines[i].remove();
  }
}


// https://davidwalsh.name/javascript-debounce-function

// Returns a function, that, as long as it continues to be invoked, will not
// be triggered. The function will be called after it stops being called for
// N milliseconds. If `immediate` is passed, trigger the function on the
// leading edge, instead of the trailing.
function debounce(func, wait, immediate) {
	var timeout;
	return function() {
		var context = this, args = arguments;
		var later = function() {
			timeout = null;
			if (!immediate) func.apply(context, args);
		};
		var callNow = immediate && !timeout;
		clearTimeout(timeout);
		timeout = setTimeout(later, wait);
		if (callNow) func.apply(context, args);
	};
};
</script>
<script>(function(global){"use strict";var undefined=void 0;var MAX_ARRAY_LENGTH=1e5;function Type(v){switch(typeof v){case"undefined":return"undefined";case"boolean":return"boolean";case"number":return"number";case"string":return"string";default:return v===null?"null":"object"}}function Class(v){return Object.prototype.toString.call(v).replace(/^\[object *|\]$/g,"")}function IsCallable(o){return typeof o==="function"}function ToObject(v){if(v===null||v===undefined)throw TypeError();return Object(v)}function ToInt32(v){return v>>0}function ToUint32(v){return v>>>0}var LN2=Math.LN2,abs=Math.abs,floor=Math.floor,log=Math.log,max=Math.max,min=Math.min,pow=Math.pow,round=Math.round;(function(){var orig=Object.defineProperty;var dom_only=!function(){try{return Object.defineProperty({},"x",{})}catch(_){return false}}();if(!orig||dom_only){Object.defineProperty=function(o,prop,desc){if(orig)try{return orig(o,prop,desc)}catch(_){}if(o!==Object(o))throw TypeError("Object.defineProperty called on non-object");if(Object.prototype.__defineGetter__&&"get"in desc)Object.prototype.__defineGetter__.call(o,prop,desc.get);if(Object.prototype.__defineSetter__&&"set"in desc)Object.prototype.__defineSetter__.call(o,prop,desc.set);if("value"in desc)o[prop]=desc.value;return o}}})();function makeArrayAccessors(obj){if(obj.length>MAX_ARRAY_LENGTH)throw RangeError("Array too large for polyfill");function makeArrayAccessor(index){Object.defineProperty(obj,index,{get:function(){return obj._getter(index)},set:function(v){obj._setter(index,v)},enumerable:true,configurable:false})}var i;for(i=0;i<obj.length;i+=1){makeArrayAccessor(i)}}function as_signed(value,bits){var s=32-bits;return value<<s>>s}function as_unsigned(value,bits){var s=32-bits;return value<<s>>>s}function packI8(n){return[n&255]}function unpackI8(bytes){return as_signed(bytes[0],8)}function packU8(n){return[n&255]}function unpackU8(bytes){return as_unsigned(bytes[0],8)}function packU8Clamped(n){n=round(Number(n));return[n<0?0:n>255?255:n&255]}function packI16(n){return[n>>8&255,n&255]}function unpackI16(bytes){return as_signed(bytes[0]<<8|bytes[1],16)}function packU16(n){return[n>>8&255,n&255]}function unpackU16(bytes){return as_unsigned(bytes[0]<<8|bytes[1],16)}function packI32(n){return[n>>24&255,n>>16&255,n>>8&255,n&255]}function unpackI32(bytes){return as_signed(bytes[0]<<24|bytes[1]<<16|bytes[2]<<8|bytes[3],32)}function packU32(n){return[n>>24&255,n>>16&255,n>>8&255,n&255]}function unpackU32(bytes){return as_unsigned(bytes[0]<<24|bytes[1]<<16|bytes[2]<<8|bytes[3],32)}function packIEEE754(v,ebits,fbits){var bias=(1<<ebits-1)-1,s,e,f,ln,i,bits,str,bytes;function roundToEven(n){var w=floor(n),f=n-w;if(f<.5)return w;if(f>.5)return w+1;return w%2?w+1:w}if(v!==v){e=(1<<ebits)-1;f=pow(2,fbits-1);s=0}else if(v===Infinity||v===-Infinity){e=(1<<ebits)-1;f=0;s=v<0?1:0}else if(v===0){e=0;f=0;s=1/v===-Infinity?1:0}else{s=v<0;v=abs(v);if(v>=pow(2,1-bias)){e=min(floor(log(v)/LN2),1023);f=roundToEven(v/pow(2,e)*pow(2,fbits));if(f/pow(2,fbits)>=2){e=e+1;f=1}if(e>bias){e=(1<<ebits)-1;f=0}else{e=e+bias;f=f-pow(2,fbits)}}else{e=0;f=roundToEven(v/pow(2,1-bias-fbits))}}bits=[];for(i=fbits;i;i-=1){bits.push(f%2?1:0);f=floor(f/2)}for(i=ebits;i;i-=1){bits.push(e%2?1:0);e=floor(e/2)}bits.push(s?1:0);bits.reverse();str=bits.join("");bytes=[];while(str.length){bytes.push(parseInt(str.substring(0,8),2));str=str.substring(8)}return bytes}function unpackIEEE754(bytes,ebits,fbits){var bits=[],i,j,b,str,bias,s,e,f;for(i=bytes.length;i;i-=1){b=bytes[i-1];for(j=8;j;j-=1){bits.push(b%2?1:0);b=b>>1}}bits.reverse();str=bits.join("");bias=(1<<ebits-1)-1;s=parseInt(str.substring(0,1),2)?-1:1;e=parseInt(str.substring(1,1+ebits),2);f=parseInt(str.substring(1+ebits),2);if(e===(1<<ebits)-1){return f!==0?NaN:s*Infinity}else if(e>0){return s*pow(2,e-bias)*(1+f/pow(2,fbits))}else if(f!==0){return s*pow(2,-(bias-1))*(f/pow(2,fbits))}else{return s<0?-0:0}}function unpackF64(b){return unpackIEEE754(b,11,52)}function packF64(v){return packIEEE754(v,11,52)}function unpackF32(b){return unpackIEEE754(b,8,23)}function packF32(v){return packIEEE754(v,8,23)}(function(){function ArrayBuffer(length){length=ToInt32(length);if(length<0)throw RangeError("ArrayBuffer size is not a small enough positive integer.");Object.defineProperty(this,"byteLength",{value:length});Object.defineProperty(this,"_bytes",{value:Array(length)});for(var i=0;i<length;i+=1)this._bytes[i]=0}global.ArrayBuffer=global.ArrayBuffer||ArrayBuffer;function $TypedArray$(){if(!arguments.length||typeof arguments[0]!=="object"){return function(length){length=ToInt32(length);if(length<0)throw RangeError("length is not a small enough positive integer.");Object.defineProperty(this,"length",{value:length});Object.defineProperty(this,"byteLength",{value:length*this.BYTES_PER_ELEMENT});Object.defineProperty(this,"buffer",{value:new ArrayBuffer(this.byteLength)});Object.defineProperty(this,"byteOffset",{value:0})}.apply(this,arguments)}if(arguments.length>=1&&Type(arguments[0])==="object"&&arguments[0]instanceof $TypedArray$){return function(typedArray){if(this.constructor!==typedArray.constructor)throw TypeError();var byteLength=typedArray.length*this.BYTES_PER_ELEMENT;Object.defineProperty(this,"buffer",{value:new ArrayBuffer(byteLength)});Object.defineProperty(this,"byteLength",{value:byteLength});Object.defineProperty(this,"byteOffset",{value:0});Object.defineProperty(this,"length",{value:typedArray.length});for(var i=0;i<this.length;i+=1)this._setter(i,typedArray._getter(i))}.apply(this,arguments)}if(arguments.length>=1&&Type(arguments[0])==="object"&&!(arguments[0]instanceof $TypedArray$)&&!(arguments[0]instanceof ArrayBuffer||Class(arguments[0])==="ArrayBuffer")){return function(array){var byteLength=array.length*this.BYTES_PER_ELEMENT;Object.defineProperty(this,"buffer",{value:new ArrayBuffer(byteLength)});Object.defineProperty(this,"byteLength",{value:byteLength});Object.defineProperty(this,"byteOffset",{value:0});Object.defineProperty(this,"length",{value:array.length});for(var i=0;i<this.length;i+=1){var s=array[i];this._setter(i,Number(s))}}.apply(this,arguments)}if(arguments.length>=1&&Type(arguments[0])==="object"&&(arguments[0]instanceof ArrayBuffer||Class(arguments[0])==="ArrayBuffer")){return function(buffer,byteOffset,length){byteOffset=ToUint32(byteOffset);if(byteOffset>buffer.byteLength)throw RangeError("byteOffset out of range");if(byteOffset%this.BYTES_PER_ELEMENT)throw RangeError("buffer length minus the byteOffset is not a multiple of the element size.");if(length===undefined){var byteLength=buffer.byteLength-byteOffset;if(byteLength%this.BYTES_PER_ELEMENT)throw RangeError("length of buffer minus byteOffset not a multiple of the element size");length=byteLength/this.BYTES_PER_ELEMENT}else{length=ToUint32(length);byteLength=length*this.BYTES_PER_ELEMENT}if(byteOffset+byteLength>buffer.byteLength)throw RangeError("byteOffset and length reference an area beyond the end of the buffer");Object.defineProperty(this,"buffer",{value:buffer});Object.defineProperty(this,"byteLength",{value:byteLength});Object.defineProperty(this,"byteOffset",{value:byteOffset});Object.defineProperty(this,"length",{value:length})}.apply(this,arguments)}throw TypeError()}Object.defineProperty($TypedArray$,"from",{value:function(iterable){return new this(iterable)}});Object.defineProperty($TypedArray$,"of",{value:function(){return new this(arguments)}});var $TypedArrayPrototype$={};$TypedArray$.prototype=$TypedArrayPrototype$;Object.defineProperty($TypedArray$.prototype,"_getter",{value:function(index){if(arguments.length<1)throw SyntaxError("Not enough arguments");index=ToUint32(index);if(index>=this.length)return undefined;var bytes=[],i,o;for(i=0,o=this.byteOffset+index*this.BYTES_PER_ELEMENT;i<this.BYTES_PER_ELEMENT;i+=1,o+=1){bytes.push(this.buffer._bytes[o])}return this._unpack(bytes)}});Object.defineProperty($TypedArray$.prototype,"get",{value:$TypedArray$.prototype._getter});Object.defineProperty($TypedArray$.prototype,"_setter",{value:function(index,value){if(arguments.length<2)throw SyntaxError("Not enough arguments");index=ToUint32(index);if(index>=this.length)return;var bytes=this._pack(value),i,o;for(i=0,o=this.byteOffset+index*this.BYTES_PER_ELEMENT;i<this.BYTES_PER_ELEMENT;i+=1,o+=1){this.buffer._bytes[o]=bytes[i]}}});Object.defineProperty($TypedArray$.prototype,"constructor",{value:$TypedArray$});Object.defineProperty($TypedArray$.prototype,"copyWithin",{value:function(target,start){var end=arguments[2];var o=ToObject(this);var lenVal=o.length;var len=ToUint32(lenVal);len=max(len,0);var relativeTarget=ToInt32(target);var to;if(relativeTarget<0)to=max(len+relativeTarget,0);else to=min(relativeTarget,len);var relativeStart=ToInt32(start);var from;if(relativeStart<0)from=max(len+relativeStart,0);else from=min(relativeStart,len);var relativeEnd;if(end===undefined)relativeEnd=len;else relativeEnd=ToInt32(end);var final;if(relativeEnd<0)final=max(len+relativeEnd,0);else final=min(relativeEnd,len);var count=min(final-from,len-to);var direction;if(from<to&&to<from+count){direction=-1;from=from+count-1;to=to+count-1}else{direction=1}while(count>0){o._setter(to,o._getter(from));from=from+direction;to=to+direction;count=count-1}return o}});Object.defineProperty($TypedArray$.prototype,"every",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();var thisArg=arguments[1];for(var i=0;i<len;i++){if(!callbackfn.call(thisArg,t._getter(i),i,t))return false}return true}});Object.defineProperty($TypedArray$.prototype,"fill",{value:function(value){var start=arguments[1],end=arguments[2];var o=ToObject(this);var lenVal=o.length;var len=ToUint32(lenVal);len=max(len,0);var relativeStart=ToInt32(start);var k;if(relativeStart<0)k=max(len+relativeStart,0);else k=min(relativeStart,len);var relativeEnd;if(end===undefined)relativeEnd=len;else relativeEnd=ToInt32(end);var final;if(relativeEnd<0)final=max(len+relativeEnd,0);else final=min(relativeEnd,len);while(k<final){o._setter(k,value);k+=1}return o}});Object.defineProperty($TypedArray$.prototype,"filter",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();var res=[];var thisp=arguments[1];for(var i=0;i<len;i++){var val=t._getter(i);if(callbackfn.call(thisp,val,i,t))res.push(val)}return new this.constructor(res)}});Object.defineProperty($TypedArray$.prototype,"find",{value:function(predicate){var o=ToObject(this);var lenValue=o.length;var len=ToUint32(lenValue);if(!IsCallable(predicate))throw TypeError();var t=arguments.length>1?arguments[1]:undefined;var k=0;while(k<len){var kValue=o._getter(k);var testResult=predicate.call(t,kValue,k,o);if(Boolean(testResult))return kValue;++k}return undefined}});Object.defineProperty($TypedArray$.prototype,"findIndex",{value:function(predicate){var o=ToObject(this);var lenValue=o.length;var len=ToUint32(lenValue);if(!IsCallable(predicate))throw TypeError();var t=arguments.length>1?arguments[1]:undefined;var k=0;while(k<len){var kValue=o._getter(k);var testResult=predicate.call(t,kValue,k,o);if(Boolean(testResult))return k;++k}return-1}});Object.defineProperty($TypedArray$.prototype,"forEach",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();var thisp=arguments[1];for(var i=0;i<len;i++)callbackfn.call(thisp,t._getter(i),i,t)}});Object.defineProperty($TypedArray$.prototype,"indexOf",{value:function(searchElement){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(len===0)return-1;var n=0;if(arguments.length>0){n=Number(arguments[1]);if(n!==n){n=0}else if(n!==0&&n!==1/0&&n!==-(1/0)){n=(n>0||-1)*floor(abs(n))}}if(n>=len)return-1;var k=n>=0?n:max(len-abs(n),0);for(;k<len;k++){if(t._getter(k)===searchElement){return k}}return-1}});Object.defineProperty($TypedArray$.prototype,"join",{value:function(separator){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);var tmp=Array(len);for(var i=0;i<len;++i)tmp[i]=t._getter(i);return tmp.join(separator===undefined?",":separator)}});Object.defineProperty($TypedArray$.prototype,"lastIndexOf",{value:function(searchElement){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(len===0)return-1;var n=len;if(arguments.length>1){n=Number(arguments[1]);if(n!==n){n=0}else if(n!==0&&n!==1/0&&n!==-(1/0)){n=(n>0||-1)*floor(abs(n))}}var k=n>=0?min(n,len-1):len-abs(n);for(;k>=0;k--){if(t._getter(k)===searchElement)return k}return-1}});Object.defineProperty($TypedArray$.prototype,"map",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();var res=[];res.length=len;var thisp=arguments[1];for(var i=0;i<len;i++)res[i]=callbackfn.call(thisp,t._getter(i),i,t);return new this.constructor(res)}});Object.defineProperty($TypedArray$.prototype,"reduce",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();if(len===0&&arguments.length===1)throw TypeError();var k=0;var accumulator;if(arguments.length>=2){accumulator=arguments[1]}else{accumulator=t._getter(k++)}while(k<len){accumulator=callbackfn.call(undefined,accumulator,t._getter(k),k,t);k++}return accumulator}});Object.defineProperty($TypedArray$.prototype,"reduceRight",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();if(len===0&&arguments.length===1)throw TypeError();var k=len-1;var accumulator;if(arguments.length>=2){accumulator=arguments[1]}else{accumulator=t._getter(k--)}while(k>=0){accumulator=callbackfn.call(undefined,accumulator,t._getter(k),k,t);k--}return accumulator}});Object.defineProperty($TypedArray$.prototype,"reverse",{value:function(){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);var half=floor(len/2);for(var i=0,j=len-1;i<half;++i,--j){var tmp=t._getter(i);t._setter(i,t._getter(j));t._setter(j,tmp)}return t}});Object.defineProperty($TypedArray$.prototype,"set",{value:function(index,value){if(arguments.length<1)throw SyntaxError("Not enough arguments");var array,sequence,offset,len,i,s,d,byteOffset,byteLength,tmp;if(typeof arguments[0]==="object"&&arguments[0].constructor===this.constructor){array=arguments[0];offset=ToUint32(arguments[1]);if(offset+array.length>this.length){throw RangeError("Offset plus length of array is out of range")}byteOffset=this.byteOffset+offset*this.BYTES_PER_ELEMENT;byteLength=array.length*this.BYTES_PER_ELEMENT;if(array.buffer===this.buffer){tmp=[];for(i=0,s=array.byteOffset;i<byteLength;i+=1,s+=1){tmp[i]=array.buffer._bytes[s]}for(i=0,d=byteOffset;i<byteLength;i+=1,d+=1){this.buffer._bytes[d]=tmp[i]}}else{for(i=0,s=array.byteOffset,d=byteOffset;i<byteLength;i+=1,s+=1,d+=1){this.buffer._bytes[d]=array.buffer._bytes[s]}}}else if(typeof arguments[0]==="object"&&typeof arguments[0].length!=="undefined"){sequence=arguments[0];len=ToUint32(sequence.length);offset=ToUint32(arguments[1]);if(offset+len>this.length){throw RangeError("Offset plus length of array is out of range")}for(i=0;i<len;i+=1){s=sequence[i];this._setter(offset+i,Number(s))}}else{throw TypeError("Unexpected argument type(s)")}}});Object.defineProperty($TypedArray$.prototype,"slice",{value:function(start,end){var o=ToObject(this);var lenVal=o.length;var len=ToUint32(lenVal);var relativeStart=ToInt32(start);var k=relativeStart<0?max(len+relativeStart,0):min(relativeStart,len);var relativeEnd=end===undefined?len:ToInt32(end);var final=relativeEnd<0?max(len+relativeEnd,0):min(relativeEnd,len);var count=final-k;var c=o.constructor;var a=new c(count);var n=0;while(k<final){var kValue=o._getter(k);a._setter(n,kValue);++k;++n}return a}});Object.defineProperty($TypedArray$.prototype,"some",{value:function(callbackfn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);if(!IsCallable(callbackfn))throw TypeError();var thisp=arguments[1];for(var i=0;i<len;i++){if(callbackfn.call(thisp,t._getter(i),i,t)){return true}}return false}});Object.defineProperty($TypedArray$.prototype,"sort",{value:function(comparefn){if(this===undefined||this===null)throw TypeError();var t=Object(this);var len=ToUint32(t.length);var tmp=Array(len);for(var i=0;i<len;++i)tmp[i]=t._getter(i);if(comparefn)tmp.sort(comparefn);else tmp.sort();for(i=0;i<len;++i)t._setter(i,tmp[i]);return t}});Object.defineProperty($TypedArray$.prototype,"subarray",{value:function(start,end){function clamp(v,min,max){return v<min?min:v>max?max:v}start=ToInt32(start);end=ToInt32(end);if(arguments.length<1){start=0}if(arguments.length<2){end=this.length}if(start<0){start=this.length+start}if(end<0){end=this.length+end}start=clamp(start,0,this.length);end=clamp(end,0,this.length);var len=end-start;if(len<0){len=0}return new this.constructor(this.buffer,this.byteOffset+start*this.BYTES_PER_ELEMENT,len)}});function makeTypedArray(elementSize,pack,unpack){var TypedArray=function(){Object.defineProperty(this,"constructor",{value:TypedArray});$TypedArray$.apply(this,arguments);makeArrayAccessors(this)};if("__proto__"in TypedArray){TypedArray.__proto__=$TypedArray$}else{TypedArray.from=$TypedArray$.from;TypedArray.of=$TypedArray$.of}TypedArray.BYTES_PER_ELEMENT=elementSize;var TypedArrayPrototype=function(){};TypedArrayPrototype.prototype=$TypedArrayPrototype$;TypedArray.prototype=new TypedArrayPrototype;Object.defineProperty(TypedArray.prototype,"BYTES_PER_ELEMENT",{value:elementSize});Object.defineProperty(TypedArray.prototype,"_pack",{value:pack});Object.defineProperty(TypedArray.prototype,"_unpack",{value:unpack});return TypedArray}var Int8Array=makeTypedArray(1,packI8,unpackI8);var Uint8Array=makeTypedArray(1,packU8,unpackU8);var Uint8ClampedArray=makeTypedArray(1,packU8Clamped,unpackU8);var Int16Array=makeTypedArray(2,packI16,unpackI16);var Uint16Array=makeTypedArray(2,packU16,unpackU16);var Int32Array=makeTypedArray(4,packI32,unpackI32);var Uint32Array=makeTypedArray(4,packU32,unpackU32);var Float32Array=makeTypedArray(4,packF32,unpackF32);var Float64Array=makeTypedArray(8,packF64,unpackF64);global.Int8Array=global.Int8Array||Int8Array;global.Uint8Array=global.Uint8Array||Uint8Array;global.Uint8ClampedArray=global.Uint8ClampedArray||Uint8ClampedArray;global.Int16Array=global.Int16Array||Int16Array;global.Uint16Array=global.Uint16Array||Uint16Array;global.Int32Array=global.Int32Array||Int32Array;global.Uint32Array=global.Uint32Array||Uint32Array;global.Float32Array=global.Float32Array||Float32Array;global.Float64Array=global.Float64Array||Float64Array})();(function(){function r(array,index){return IsCallable(array.get)?array.get(index):array[index]}var IS_BIG_ENDIAN=function(){var u16array=new Uint16Array([4660]),u8array=new Uint8Array(u16array.buffer);return r(u8array,0)===18}();function DataView(buffer,byteOffset,byteLength){if(!(buffer instanceof ArrayBuffer||Class(buffer)==="ArrayBuffer"))throw TypeError();byteOffset=ToUint32(byteOffset);if(byteOffset>buffer.byteLength)throw RangeError("byteOffset out of range");if(byteLength===undefined)byteLength=buffer.byteLength-byteOffset;else byteLength=ToUint32(byteLength);if(byteOffset+byteLength>buffer.byteLength)throw RangeError("byteOffset and length reference an area beyond the end of the buffer");Object.defineProperty(this,"buffer",{value:buffer});Object.defineProperty(this,"byteLength",{value:byteLength});Object.defineProperty(this,"byteOffset",{value:byteOffset})}function makeGetter(arrayType){return function GetViewValue(byteOffset,littleEndian){byteOffset=ToUint32(byteOffset);if(byteOffset+arrayType.BYTES_PER_ELEMENT>this.byteLength)throw RangeError("Array index out of range");byteOffset+=this.byteOffset;var uint8Array=new Uint8Array(this.buffer,byteOffset,arrayType.BYTES_PER_ELEMENT),bytes=[];for(var i=0;i<arrayType.BYTES_PER_ELEMENT;i+=1)bytes.push(r(uint8Array,i));if(Boolean(littleEndian)===Boolean(IS_BIG_ENDIAN))bytes.reverse();return r(new arrayType(new Uint8Array(bytes).buffer),0)}}Object.defineProperty(DataView.prototype,"getUint8",{value:makeGetter(Uint8Array)});Object.defineProperty(DataView.prototype,"getInt8",{value:makeGetter(Int8Array)});Object.defineProperty(DataView.prototype,"getUint16",{value:makeGetter(Uint16Array)});Object.defineProperty(DataView.prototype,"getInt16",{value:makeGetter(Int16Array)});Object.defineProperty(DataView.prototype,"getUint32",{value:makeGetter(Uint32Array)});Object.defineProperty(DataView.prototype,"getInt32",{value:makeGetter(Int32Array)});Object.defineProperty(DataView.prototype,"getFloat32",{value:makeGetter(Float32Array)});Object.defineProperty(DataView.prototype,"getFloat64",{value:makeGetter(Float64Array)});function makeSetter(arrayType){return function SetViewValue(byteOffset,value,littleEndian){byteOffset=ToUint32(byteOffset);if(byteOffset+arrayType.BYTES_PER_ELEMENT>this.byteLength)throw RangeError("Array index out of range");var typeArray=new arrayType([value]),byteArray=new Uint8Array(typeArray.buffer),bytes=[],i,byteView;for(i=0;i<arrayType.BYTES_PER_ELEMENT;i+=1)bytes.push(r(byteArray,i));if(Boolean(littleEndian)===Boolean(IS_BIG_ENDIAN))bytes.reverse();byteView=new Uint8Array(this.buffer,byteOffset,arrayType.BYTES_PER_ELEMENT);byteView.set(bytes)}}Object.defineProperty(DataView.prototype,"setUint8",{value:makeSetter(Uint8Array)});Object.defineProperty(DataView.prototype,"setInt8",{value:makeSetter(Int8Array)});Object.defineProperty(DataView.prototype,"setUint16",{value:makeSetter(Uint16Array)});Object.defineProperty(DataView.prototype,"setInt16",{value:makeSetter(Int16Array)});Object.defineProperty(DataView.prototype,"setUint32",{value:makeSetter(Uint32Array)});Object.defineProperty(DataView.prototype,"setInt32",{value:makeSetter(Int32Array)});Object.defineProperty(DataView.prototype,"setFloat32",{value:makeSetter(Float32Array)});Object.defineProperty(DataView.prototype,"setFloat64",{value:makeSetter(Float64Array)});global.DataView=global.DataView||DataView})()})(this);</script>
<script>/*! jQuery v1.11.3 | (c) 2005, 2015 jQuery Foundation, Inc. | jquery.org/license */
!function(a,b){"object"==typeof module&&"object"==typeof module.exports?module.exports=a.document?b(a,!0):function(a){if(!a.document)throw new Error("jQuery requires a window with a document");return b(a)}:b(a)}("undefined"!=typeof window?window:this,function(a,b){var c=[],d=c.slice,e=c.concat,f=c.push,g=c.indexOf,h={},i=h.toString,j=h.hasOwnProperty,k={},l="1.11.3",m=function(a,b){return new m.fn.init(a,b)},n=/^[\s\uFEFF\xA0]+|[\s\uFEFF\xA0]+$/g,o=/^-ms-/,p=/-([\da-z])/gi,q=function(a,b){return b.toUpperCase()};m.fn=m.prototype={jquery:l,constructor:m,selector:"",length:0,toArray:function(){return d.call(this)},get:function(a){return null!=a?0>a?this[a+this.length]:this[a]:d.call(this)},pushStack:function(a){var b=m.merge(this.constructor(),a);return b.prevObject=this,b.context=this.context,b},each:function(a,b){return m.each(this,a,b)},map:function(a){return this.pushStack(m.map(this,function(b,c){return a.call(b,c,b)}))},slice:function(){return this.pushStack(d.apply(this,arguments))},first:function(){return this.eq(0)},last:function(){return this.eq(-1)},eq:function(a){var b=this.length,c=+a+(0>a?b:0);return this.pushStack(c>=0&&b>c?[this[c]]:[])},end:function(){return this.prevObject||this.constructor(null)},push:f,sort:c.sort,splice:c.splice},m.extend=m.fn.extend=function(){var a,b,c,d,e,f,g=arguments[0]||{},h=1,i=arguments.length,j=!1;for("boolean"==typeof g&&(j=g,g=arguments[h]||{},h++),"object"==typeof g||m.isFunction(g)||(g={}),h===i&&(g=this,h--);i>h;h++)if(null!=(e=arguments[h]))for(d in e)a=g[d],c=e[d],g!==c&&(j&&c&&(m.isPlainObject(c)||(b=m.isArray(c)))?(b?(b=!1,f=a&&m.isArray(a)?a:[]):f=a&&m.isPlainObject(a)?a:{},g[d]=m.extend(j,f,c)):void 0!==c&&(g[d]=c));return g},m.extend({expando:"jQuery"+(l+Math.random()).replace(/\D/g,""),isReady:!0,error:function(a){throw new Error(a)},noop:function(){},isFunction:function(a){return"function"===m.type(a)},isArray:Array.isArray||function(a){return"array"===m.type(a)},isWindow:function(a){return null!=a&&a==a.window},isNumeric:function(a){return!m.isArray(a)&&a-parseFloat(a)+1>=0},isEmptyObject:function(a){var b;for(b in a)return!1;return!0},isPlainObject:function(a){var b;if(!a||"object"!==m.type(a)||a.nodeType||m.isWindow(a))return!1;try{if(a.constructor&&!j.call(a,"constructor")&&!j.call(a.constructor.prototype,"isPrototypeOf"))return!1}catch(c){return!1}if(k.ownLast)for(b in a)return j.call(a,b);for(b in a);return void 0===b||j.call(a,b)},type:function(a){return null==a?a+"":"object"==typeof a||"function"==typeof a?h[i.call(a)]||"object":typeof a},globalEval:function(b){b&&m.trim(b)&&(a.execScript||function(b){a.eval.call(a,b)})(b)},camelCase:function(a){return a.replace(o,"ms-").replace(p,q)},nodeName:function(a,b){return a.nodeName&&a.nodeName.toLowerCase()===b.toLowerCase()},each:function(a,b,c){var d,e=0,f=a.length,g=r(a);if(c){if(g){for(;f>e;e++)if(d=b.apply(a[e],c),d===!1)break}else for(e in a)if(d=b.apply(a[e],c),d===!1)break}else if(g){for(;f>e;e++)if(d=b.call(a[e],e,a[e]),d===!1)break}else for(e in a)if(d=b.call(a[e],e,a[e]),d===!1)break;return a},trim:function(a){return null==a?"":(a+"").replace(n,"")},makeArray:function(a,b){var c=b||[];return null!=a&&(r(Object(a))?m.merge(c,"string"==typeof a?[a]:a):f.call(c,a)),c},inArray:function(a,b,c){var d;if(b){if(g)return g.call(b,a,c);for(d=b.length,c=c?0>c?Math.max(0,d+c):c:0;d>c;c++)if(c in b&&b[c]===a)return c}return-1},merge:function(a,b){var c=+b.length,d=0,e=a.length;while(c>d)a[e++]=b[d++];if(c!==c)while(void 0!==b[d])a[e++]=b[d++];return a.length=e,a},grep:function(a,b,c){for(var d,e=[],f=0,g=a.length,h=!c;g>f;f++)d=!b(a[f],f),d!==h&&e.push(a[f]);return e},map:function(a,b,c){var d,f=0,g=a.length,h=r(a),i=[];if(h)for(;g>f;f++)d=b(a[f],f,c),null!=d&&i.push(d);else for(f in a)d=b(a[f],f,c),null!=d&&i.push(d);return e.apply([],i)},guid:1,proxy:function(a,b){var c,e,f;return"string"==typeof b&&(f=a[b],b=a,a=f),m.isFunction(a)?(c=d.call(arguments,2),e=function(){return a.apply(b||this,c.concat(d.call(arguments)))},e.guid=a.guid=a.guid||m.guid++,e):void 0},now:function(){return+new Date},support:k}),m.each("Boolean Number String Function Array Date RegExp Object Error".split(" "),function(a,b){h["[object "+b+"]"]=b.toLowerCase()});function r(a){var b="length"in a&&a.length,c=m.type(a);return"function"===c||m.isWindow(a)?!1:1===a.nodeType&&b?!0:"array"===c||0===b||"number"==typeof b&&b>0&&b-1 in a}var s=function(a){var b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u="sizzle"+1*new Date,v=a.document,w=0,x=0,y=ha(),z=ha(),A=ha(),B=function(a,b){return a===b&&(l=!0),0},C=1<<31,D={}.hasOwnProperty,E=[],F=E.pop,G=E.push,H=E.push,I=E.slice,J=function(a,b){for(var c=0,d=a.length;d>c;c++)if(a[c]===b)return c;return-1},K="checked|selected|async|autofocus|autoplay|controls|defer|disabled|hidden|ismap|loop|multiple|open|readonly|required|scoped",L="[\\x20\\t\\r\\n\\f]",M="(?:\\\\.|[\\w-]|[^\\x00-\\xa0])+",N=M.replace("w","w#"),O="\\["+L+"*("+M+")(?:"+L+"*([*^$|!~]?=)"+L+"*(?:'((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\"|("+N+"))|)"+L+"*\\]",P=":("+M+")(?:\\((('((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\")|((?:\\\\.|[^\\\\()[\\]]|"+O+")*)|.*)\\)|)",Q=new RegExp(L+"+","g"),R=new RegExp("^"+L+"+|((?:^|[^\\\\])(?:\\\\.)*)"+L+"+$","g"),S=new RegExp("^"+L+"*,"+L+"*"),T=new RegExp("^"+L+"*([>+~]|"+L+")"+L+"*"),U=new RegExp("="+L+"*([^\\]'\"]*?)"+L+"*\\]","g"),V=new RegExp(P),W=new RegExp("^"+N+"$"),X={ID:new RegExp("^#("+M+")"),CLASS:new RegExp("^\\.("+M+")"),TAG:new RegExp("^("+M.replace("w","w*")+")"),ATTR:new RegExp("^"+O),PSEUDO:new RegExp("^"+P),CHILD:new RegExp("^:(only|first|last|nth|nth-last)-(child|of-type)(?:\\("+L+"*(even|odd|(([+-]|)(\\d*)n|)"+L+"*(?:([+-]|)"+L+"*(\\d+)|))"+L+"*\\)|)","i"),bool:new RegExp("^(?:"+K+")$","i"),needsContext:new RegExp("^"+L+"*[>+~]|:(even|odd|eq|gt|lt|nth|first|last)(?:\\("+L+"*((?:-\\d)?\\d*)"+L+"*\\)|)(?=[^-]|$)","i")},Y=/^(?:input|select|textarea|button)$/i,Z=/^h\d$/i,$=/^[^{]+\{\s*\[native \w/,_=/^(?:#([\w-]+)|(\w+)|\.([\w-]+))$/,aa=/[+~]/,ba=/'|\\/g,ca=new RegExp("\\\\([\\da-f]{1,6}"+L+"?|("+L+")|.)","ig"),da=function(a,b,c){var d="0x"+b-65536;return d!==d||c?b:0>d?String.fromCharCode(d+65536):String.fromCharCode(d>>10|55296,1023&d|56320)},ea=function(){m()};try{H.apply(E=I.call(v.childNodes),v.childNodes),E[v.childNodes.length].nodeType}catch(fa){H={apply:E.length?function(a,b){G.apply(a,I.call(b))}:function(a,b){var c=a.length,d=0;while(a[c++]=b[d++]);a.length=c-1}}}function ga(a,b,d,e){var f,h,j,k,l,o,r,s,w,x;if((b?b.ownerDocument||b:v)!==n&&m(b),b=b||n,d=d||[],k=b.nodeType,"string"!=typeof a||!a||1!==k&&9!==k&&11!==k)return d;if(!e&&p){if(11!==k&&(f=_.exec(a)))if(j=f[1]){if(9===k){if(h=b.getElementById(j),!h||!h.parentNode)return d;if(h.id===j)return d.push(h),d}else if(b.ownerDocument&&(h=b.ownerDocument.getElementById(j))&&t(b,h)&&h.id===j)return d.push(h),d}else{if(f[2])return H.apply(d,b.getElementsByTagName(a)),d;if((j=f[3])&&c.getElementsByClassName)return H.apply(d,b.getElementsByClassName(j)),d}if(c.qsa&&(!q||!q.test(a))){if(s=r=u,w=b,x=1!==k&&a,1===k&&"object"!==b.nodeName.toLowerCase()){o=g(a),(r=b.getAttribute("id"))?s=r.replace(ba,"\\$&"):b.setAttribute("id",s),s="[id='"+s+"'] ",l=o.length;while(l--)o[l]=s+ra(o[l]);w=aa.test(a)&&pa(b.parentNode)||b,x=o.join(",")}if(x)try{return H.apply(d,w.querySelectorAll(x)),d}catch(y){}finally{r||b.removeAttribute("id")}}}return i(a.replace(R,"$1"),b,d,e)}function ha(){var a=[];function b(c,e){return a.push(c+" ")>d.cacheLength&&delete b[a.shift()],b[c+" "]=e}return b}function ia(a){return a[u]=!0,a}function ja(a){var b=n.createElement("div");try{return!!a(b)}catch(c){return!1}finally{b.parentNode&&b.parentNode.removeChild(b),b=null}}function ka(a,b){var c=a.split("|"),e=a.length;while(e--)d.attrHandle[c[e]]=b}function la(a,b){var c=b&&a,d=c&&1===a.nodeType&&1===b.nodeType&&(~b.sourceIndex||C)-(~a.sourceIndex||C);if(d)return d;if(c)while(c=c.nextSibling)if(c===b)return-1;return a?1:-1}function ma(a){return function(b){var c=b.nodeName.toLowerCase();return"input"===c&&b.type===a}}function na(a){return function(b){var c=b.nodeName.toLowerCase();return("input"===c||"button"===c)&&b.type===a}}function oa(a){return ia(function(b){return b=+b,ia(function(c,d){var e,f=a([],c.length,b),g=f.length;while(g--)c[e=f[g]]&&(c[e]=!(d[e]=c[e]))})})}function pa(a){return a&&"undefined"!=typeof a.getElementsByTagName&&a}c=ga.support={},f=ga.isXML=function(a){var b=a&&(a.ownerDocument||a).documentElement;return b?"HTML"!==b.nodeName:!1},m=ga.setDocument=function(a){var b,e,g=a?a.ownerDocument||a:v;return g!==n&&9===g.nodeType&&g.documentElement?(n=g,o=g.documentElement,e=g.defaultView,e&&e!==e.top&&(e.addEventListener?e.addEventListener("unload",ea,!1):e.attachEvent&&e.attachEvent("onunload",ea)),p=!f(g),c.attributes=ja(function(a){return a.className="i",!a.getAttribute("className")}),c.getElementsByTagName=ja(function(a){return a.appendChild(g.createComment("")),!a.getElementsByTagName("*").length}),c.getElementsByClassName=$.test(g.getElementsByClassName),c.getById=ja(function(a){return o.appendChild(a).id=u,!g.getElementsByName||!g.getElementsByName(u).length}),c.getById?(d.find.ID=function(a,b){if("undefined"!=typeof b.getElementById&&p){var c=b.getElementById(a);return c&&c.parentNode?[c]:[]}},d.filter.ID=function(a){var b=a.replace(ca,da);return function(a){return a.getAttribute("id")===b}}):(delete d.find.ID,d.filter.ID=function(a){var b=a.replace(ca,da);return function(a){var c="undefined"!=typeof a.getAttributeNode&&a.getAttributeNode("id");return c&&c.value===b}}),d.find.TAG=c.getElementsByTagName?function(a,b){return"undefined"!=typeof b.getElementsByTagName?b.getElementsByTagName(a):c.qsa?b.querySelectorAll(a):void 0}:function(a,b){var c,d=[],e=0,f=b.getElementsByTagName(a);if("*"===a){while(c=f[e++])1===c.nodeType&&d.push(c);return d}return f},d.find.CLASS=c.getElementsByClassName&&function(a,b){return p?b.getElementsByClassName(a):void 0},r=[],q=[],(c.qsa=$.test(g.querySelectorAll))&&(ja(function(a){o.appendChild(a).innerHTML="<a id='"+u+"'></a><select id='"+u+"-\f]' msallowcapture=''><option selected=''></option></select>",a.querySelectorAll("[msallowcapture^='']").length&&q.push("[*^$]="+L+"*(?:''|\"\")"),a.querySelectorAll("[selected]").length||q.push("\\["+L+"*(?:value|"+K+")"),a.querySelectorAll("[id~="+u+"-]").length||q.push("~="),a.querySelectorAll(":checked").length||q.push(":checked"),a.querySelectorAll("a#"+u+"+*").length||q.push(".#.+[+~]")}),ja(function(a){var b=g.createElement("input");b.setAttribute("type","hidden"),a.appendChild(b).setAttribute("name","D"),a.querySelectorAll("[name=d]").length&&q.push("name"+L+"*[*^$|!~]?="),a.querySelectorAll(":enabled").length||q.push(":enabled",":disabled"),a.querySelectorAll("*,:x"),q.push(",.*:")})),(c.matchesSelector=$.test(s=o.matches||o.webkitMatchesSelector||o.mozMatchesSelector||o.oMatchesSelector||o.msMatchesSelector))&&ja(function(a){c.disconnectedMatch=s.call(a,"div"),s.call(a,"[s!='']:x"),r.push("!=",P)}),q=q.length&&new RegExp(q.join("|")),r=r.length&&new RegExp(r.join("|")),b=$.test(o.compareDocumentPosition),t=b||$.test(o.contains)?function(a,b){var c=9===a.nodeType?a.documentElement:a,d=b&&b.parentNode;return a===d||!(!d||1!==d.nodeType||!(c.contains?c.contains(d):a.compareDocumentPosition&&16&a.compareDocumentPosition(d)))}:function(a,b){if(b)while(b=b.parentNode)if(b===a)return!0;return!1},B=b?function(a,b){if(a===b)return l=!0,0;var d=!a.compareDocumentPosition-!b.compareDocumentPosition;return d?d:(d=(a.ownerDocument||a)===(b.ownerDocument||b)?a.compareDocumentPosition(b):1,1&d||!c.sortDetached&&b.compareDocumentPosition(a)===d?a===g||a.ownerDocument===v&&t(v,a)?-1:b===g||b.ownerDocument===v&&t(v,b)?1:k?J(k,a)-J(k,b):0:4&d?-1:1)}:function(a,b){if(a===b)return l=!0,0;var c,d=0,e=a.parentNode,f=b.parentNode,h=[a],i=[b];if(!e||!f)return a===g?-1:b===g?1:e?-1:f?1:k?J(k,a)-J(k,b):0;if(e===f)return la(a,b);c=a;while(c=c.parentNode)h.unshift(c);c=b;while(c=c.parentNode)i.unshift(c);while(h[d]===i[d])d++;return d?la(h[d],i[d]):h[d]===v?-1:i[d]===v?1:0},g):n},ga.matches=function(a,b){return ga(a,null,null,b)},ga.matchesSelector=function(a,b){if((a.ownerDocument||a)!==n&&m(a),b=b.replace(U,"='$1']"),!(!c.matchesSelector||!p||r&&r.test(b)||q&&q.test(b)))try{var d=s.call(a,b);if(d||c.disconnectedMatch||a.document&&11!==a.document.nodeType)return d}catch(e){}return ga(b,n,null,[a]).length>0},ga.contains=function(a,b){return(a.ownerDocument||a)!==n&&m(a),t(a,b)},ga.attr=function(a,b){(a.ownerDocument||a)!==n&&m(a);var e=d.attrHandle[b.toLowerCase()],f=e&&D.call(d.attrHandle,b.toLowerCase())?e(a,b,!p):void 0;return void 0!==f?f:c.attributes||!p?a.getAttribute(b):(f=a.getAttributeNode(b))&&f.specified?f.value:null},ga.error=function(a){throw new Error("Syntax error, unrecognized expression: "+a)},ga.uniqueSort=function(a){var b,d=[],e=0,f=0;if(l=!c.detectDuplicates,k=!c.sortStable&&a.slice(0),a.sort(B),l){while(b=a[f++])b===a[f]&&(e=d.push(f));while(e--)a.splice(d[e],1)}return k=null,a},e=ga.getText=function(a){var b,c="",d=0,f=a.nodeType;if(f){if(1===f||9===f||11===f){if("string"==typeof a.textContent)return a.textContent;for(a=a.firstChild;a;a=a.nextSibling)c+=e(a)}else if(3===f||4===f)return a.nodeValue}else while(b=a[d++])c+=e(b);return c},d=ga.selectors={cacheLength:50,createPseudo:ia,match:X,attrHandle:{},find:{},relative:{">":{dir:"parentNode",first:!0}," ":{dir:"parentNode"},"+":{dir:"previousSibling",first:!0},"~":{dir:"previousSibling"}},preFilter:{ATTR:function(a){return a[1]=a[1].replace(ca,da),a[3]=(a[3]||a[4]||a[5]||"").replace(ca,da),"~="===a[2]&&(a[3]=" "+a[3]+" "),a.slice(0,4)},CHILD:function(a){return a[1]=a[1].toLowerCase(),"nth"===a[1].slice(0,3)?(a[3]||ga.error(a[0]),a[4]=+(a[4]?a[5]+(a[6]||1):2*("even"===a[3]||"odd"===a[3])),a[5]=+(a[7]+a[8]||"odd"===a[3])):a[3]&&ga.error(a[0]),a},PSEUDO:function(a){var b,c=!a[6]&&a[2];return X.CHILD.test(a[0])?null:(a[3]?a[2]=a[4]||a[5]||"":c&&V.test(c)&&(b=g(c,!0))&&(b=c.indexOf(")",c.length-b)-c.length)&&(a[0]=a[0].slice(0,b),a[2]=c.slice(0,b)),a.slice(0,3))}},filter:{TAG:function(a){var b=a.replace(ca,da).toLowerCase();return"*"===a?function(){return!0}:function(a){return a.nodeName&&a.nodeName.toLowerCase()===b}},CLASS:function(a){var b=y[a+" "];return b||(b=new RegExp("(^|"+L+")"+a+"("+L+"|$)"))&&y(a,function(a){return b.test("string"==typeof a.className&&a.className||"undefined"!=typeof a.getAttribute&&a.getAttribute("class")||"")})},ATTR:function(a,b,c){return function(d){var e=ga.attr(d,a);return null==e?"!="===b:b?(e+="","="===b?e===c:"!="===b?e!==c:"^="===b?c&&0===e.indexOf(c):"*="===b?c&&e.indexOf(c)>-1:"$="===b?c&&e.slice(-c.length)===c:"~="===b?(" "+e.replace(Q," ")+" ").indexOf(c)>-1:"|="===b?e===c||e.slice(0,c.length+1)===c+"-":!1):!0}},CHILD:function(a,b,c,d,e){var f="nth"!==a.slice(0,3),g="last"!==a.slice(-4),h="of-type"===b;return 1===d&&0===e?function(a){return!!a.parentNode}:function(b,c,i){var j,k,l,m,n,o,p=f!==g?"nextSibling":"previousSibling",q=b.parentNode,r=h&&b.nodeName.toLowerCase(),s=!i&&!h;if(q){if(f){while(p){l=b;while(l=l[p])if(h?l.nodeName.toLowerCase()===r:1===l.nodeType)return!1;o=p="only"===a&&!o&&"nextSibling"}return!0}if(o=[g?q.firstChild:q.lastChild],g&&s){k=q[u]||(q[u]={}),j=k[a]||[],n=j[0]===w&&j[1],m=j[0]===w&&j[2],l=n&&q.childNodes[n];while(l=++n&&l&&l[p]||(m=n=0)||o.pop())if(1===l.nodeType&&++m&&l===b){k[a]=[w,n,m];break}}else if(s&&(j=(b[u]||(b[u]={}))[a])&&j[0]===w)m=j[1];else while(l=++n&&l&&l[p]||(m=n=0)||o.pop())if((h?l.nodeName.toLowerCase()===r:1===l.nodeType)&&++m&&(s&&((l[u]||(l[u]={}))[a]=[w,m]),l===b))break;return m-=e,m===d||m%d===0&&m/d>=0}}},PSEUDO:function(a,b){var c,e=d.pseudos[a]||d.setFilters[a.toLowerCase()]||ga.error("unsupported pseudo: "+a);return e[u]?e(b):e.length>1?(c=[a,a,"",b],d.setFilters.hasOwnProperty(a.toLowerCase())?ia(function(a,c){var d,f=e(a,b),g=f.length;while(g--)d=J(a,f[g]),a[d]=!(c[d]=f[g])}):function(a){return e(a,0,c)}):e}},pseudos:{not:ia(function(a){var b=[],c=[],d=h(a.replace(R,"$1"));return d[u]?ia(function(a,b,c,e){var f,g=d(a,null,e,[]),h=a.length;while(h--)(f=g[h])&&(a[h]=!(b[h]=f))}):function(a,e,f){return b[0]=a,d(b,null,f,c),b[0]=null,!c.pop()}}),has:ia(function(a){return function(b){return ga(a,b).length>0}}),contains:ia(function(a){return a=a.replace(ca,da),function(b){return(b.textContent||b.innerText||e(b)).indexOf(a)>-1}}),lang:ia(function(a){return W.test(a||"")||ga.error("unsupported lang: "+a),a=a.replace(ca,da).toLowerCase(),function(b){var c;do if(c=p?b.lang:b.getAttribute("xml:lang")||b.getAttribute("lang"))return c=c.toLowerCase(),c===a||0===c.indexOf(a+"-");while((b=b.parentNode)&&1===b.nodeType);return!1}}),target:function(b){var c=a.location&&a.location.hash;return c&&c.slice(1)===b.id},root:function(a){return a===o},focus:function(a){return a===n.activeElement&&(!n.hasFocus||n.hasFocus())&&!!(a.type||a.href||~a.tabIndex)},enabled:function(a){return a.disabled===!1},disabled:function(a){return a.disabled===!0},checked:function(a){var b=a.nodeName.toLowerCase();return"input"===b&&!!a.checked||"option"===b&&!!a.selected},selected:function(a){return a.parentNode&&a.parentNode.selectedIndex,a.selected===!0},empty:function(a){for(a=a.firstChild;a;a=a.nextSibling)if(a.nodeType<6)return!1;return!0},parent:function(a){return!d.pseudos.empty(a)},header:function(a){return Z.test(a.nodeName)},input:function(a){return Y.test(a.nodeName)},button:function(a){var b=a.nodeName.toLowerCase();return"input"===b&&"button"===a.type||"button"===b},text:function(a){var b;return"input"===a.nodeName.toLowerCase()&&"text"===a.type&&(null==(b=a.getAttribute("type"))||"text"===b.toLowerCase())},first:oa(function(){return[0]}),last:oa(function(a,b){return[b-1]}),eq:oa(function(a,b,c){return[0>c?c+b:c]}),even:oa(function(a,b){for(var c=0;b>c;c+=2)a.push(c);return a}),odd:oa(function(a,b){for(var c=1;b>c;c+=2)a.push(c);return a}),lt:oa(function(a,b,c){for(var d=0>c?c+b:c;--d>=0;)a.push(d);return a}),gt:oa(function(a,b,c){for(var d=0>c?c+b:c;++d<b;)a.push(d);return a})}},d.pseudos.nth=d.pseudos.eq;for(b in{radio:!0,checkbox:!0,file:!0,password:!0,image:!0})d.pseudos[b]=ma(b);for(b in{submit:!0,reset:!0})d.pseudos[b]=na(b);function qa(){}qa.prototype=d.filters=d.pseudos,d.setFilters=new qa,g=ga.tokenize=function(a,b){var c,e,f,g,h,i,j,k=z[a+" "];if(k)return b?0:k.slice(0);h=a,i=[],j=d.preFilter;while(h){(!c||(e=S.exec(h)))&&(e&&(h=h.slice(e[0].length)||h),i.push(f=[])),c=!1,(e=T.exec(h))&&(c=e.shift(),f.push({value:c,type:e[0].replace(R," ")}),h=h.slice(c.length));for(g in d.filter)!(e=X[g].exec(h))||j[g]&&!(e=j[g](e))||(c=e.shift(),f.push({value:c,type:g,matches:e}),h=h.slice(c.length));if(!c)break}return b?h.length:h?ga.error(a):z(a,i).slice(0)};function ra(a){for(var b=0,c=a.length,d="";c>b;b++)d+=a[b].value;return d}function sa(a,b,c){var d=b.dir,e=c&&"parentNode"===d,f=x++;return b.first?function(b,c,f){while(b=b[d])if(1===b.nodeType||e)return a(b,c,f)}:function(b,c,g){var h,i,j=[w,f];if(g){while(b=b[d])if((1===b.nodeType||e)&&a(b,c,g))return!0}else while(b=b[d])if(1===b.nodeType||e){if(i=b[u]||(b[u]={}),(h=i[d])&&h[0]===w&&h[1]===f)return j[2]=h[2];if(i[d]=j,j[2]=a(b,c,g))return!0}}}function ta(a){return a.length>1?function(b,c,d){var e=a.length;while(e--)if(!a[e](b,c,d))return!1;return!0}:a[0]}function ua(a,b,c){for(var d=0,e=b.length;e>d;d++)ga(a,b[d],c);return c}function va(a,b,c,d,e){for(var f,g=[],h=0,i=a.length,j=null!=b;i>h;h++)(f=a[h])&&(!c||c(f,d,e))&&(g.push(f),j&&b.push(h));return g}function wa(a,b,c,d,e,f){return d&&!d[u]&&(d=wa(d)),e&&!e[u]&&(e=wa(e,f)),ia(function(f,g,h,i){var j,k,l,m=[],n=[],o=g.length,p=f||ua(b||"*",h.nodeType?[h]:h,[]),q=!a||!f&&b?p:va(p,m,a,h,i),r=c?e||(f?a:o||d)?[]:g:q;if(c&&c(q,r,h,i),d){j=va(r,n),d(j,[],h,i),k=j.length;while(k--)(l=j[k])&&(r[n[k]]=!(q[n[k]]=l))}if(f){if(e||a){if(e){j=[],k=r.length;while(k--)(l=r[k])&&j.push(q[k]=l);e(null,r=[],j,i)}k=r.length;while(k--)(l=r[k])&&(j=e?J(f,l):m[k])>-1&&(f[j]=!(g[j]=l))}}else r=va(r===g?r.splice(o,r.length):r),e?e(null,g,r,i):H.apply(g,r)})}function xa(a){for(var b,c,e,f=a.length,g=d.relative[a[0].type],h=g||d.relative[" "],i=g?1:0,k=sa(function(a){return a===b},h,!0),l=sa(function(a){return J(b,a)>-1},h,!0),m=[function(a,c,d){var e=!g&&(d||c!==j)||((b=c).nodeType?k(a,c,d):l(a,c,d));return b=null,e}];f>i;i++)if(c=d.relative[a[i].type])m=[sa(ta(m),c)];else{if(c=d.filter[a[i].type].apply(null,a[i].matches),c[u]){for(e=++i;f>e;e++)if(d.relative[a[e].type])break;return wa(i>1&&ta(m),i>1&&ra(a.slice(0,i-1).concat({value:" "===a[i-2].type?"*":""})).replace(R,"$1"),c,e>i&&xa(a.slice(i,e)),f>e&&xa(a=a.slice(e)),f>e&&ra(a))}m.push(c)}return ta(m)}function ya(a,b){var c=b.length>0,e=a.length>0,f=function(f,g,h,i,k){var l,m,o,p=0,q="0",r=f&&[],s=[],t=j,u=f||e&&d.find.TAG("*",k),v=w+=null==t?1:Math.random()||.1,x=u.length;for(k&&(j=g!==n&&g);q!==x&&null!=(l=u[q]);q++){if(e&&l){m=0;while(o=a[m++])if(o(l,g,h)){i.push(l);break}k&&(w=v)}c&&((l=!o&&l)&&p--,f&&r.push(l))}if(p+=q,c&&q!==p){m=0;while(o=b[m++])o(r,s,g,h);if(f){if(p>0)while(q--)r[q]||s[q]||(s[q]=F.call(i));s=va(s)}H.apply(i,s),k&&!f&&s.length>0&&p+b.length>1&&ga.uniqueSort(i)}return k&&(w=v,j=t),r};return c?ia(f):f}return h=ga.compile=function(a,b){var c,d=[],e=[],f=A[a+" "];if(!f){b||(b=g(a)),c=b.length;while(c--)f=xa(b[c]),f[u]?d.push(f):e.push(f);f=A(a,ya(e,d)),f.selector=a}return f},i=ga.select=function(a,b,e,f){var i,j,k,l,m,n="function"==typeof a&&a,o=!f&&g(a=n.selector||a);if(e=e||[],1===o.length){if(j=o[0]=o[0].slice(0),j.length>2&&"ID"===(k=j[0]).type&&c.getById&&9===b.nodeType&&p&&d.relative[j[1].type]){if(b=(d.find.ID(k.matches[0].replace(ca,da),b)||[])[0],!b)return e;n&&(b=b.parentNode),a=a.slice(j.shift().value.length)}i=X.needsContext.test(a)?0:j.length;while(i--){if(k=j[i],d.relative[l=k.type])break;if((m=d.find[l])&&(f=m(k.matches[0].replace(ca,da),aa.test(j[0].type)&&pa(b.parentNode)||b))){if(j.splice(i,1),a=f.length&&ra(j),!a)return H.apply(e,f),e;break}}}return(n||h(a,o))(f,b,!p,e,aa.test(a)&&pa(b.parentNode)||b),e},c.sortStable=u.split("").sort(B).join("")===u,c.detectDuplicates=!!l,m(),c.sortDetached=ja(function(a){return 1&a.compareDocumentPosition(n.createElement("div"))}),ja(function(a){return a.innerHTML="<a href='#'></a>","#"===a.firstChild.getAttribute("href")})||ka("type|href|height|width",function(a,b,c){return c?void 0:a.getAttribute(b,"type"===b.toLowerCase()?1:2)}),c.attributes&&ja(function(a){return a.innerHTML="<input/>",a.firstChild.setAttribute("value",""),""===a.firstChild.getAttribute("value")})||ka("value",function(a,b,c){return c||"input"!==a.nodeName.toLowerCase()?void 0:a.defaultValue}),ja(function(a){return null==a.getAttribute("disabled")})||ka(K,function(a,b,c){var d;return c?void 0:a[b]===!0?b.toLowerCase():(d=a.getAttributeNode(b))&&d.specified?d.value:null}),ga}(a);m.find=s,m.expr=s.selectors,m.expr[":"]=m.expr.pseudos,m.unique=s.uniqueSort,m.text=s.getText,m.isXMLDoc=s.isXML,m.contains=s.contains;var t=m.expr.match.needsContext,u=/^<(\w+)\s*\/?>(?:<\/\1>|)$/,v=/^.[^:#\[\.,]*$/;function w(a,b,c){if(m.isFunction(b))return m.grep(a,function(a,d){return!!b.call(a,d,a)!==c});if(b.nodeType)return m.grep(a,function(a){return a===b!==c});if("string"==typeof b){if(v.test(b))return m.filter(b,a,c);b=m.filter(b,a)}return m.grep(a,function(a){return m.inArray(a,b)>=0!==c})}m.filter=function(a,b,c){var d=b[0];return c&&(a=":not("+a+")"),1===b.length&&1===d.nodeType?m.find.matchesSelector(d,a)?[d]:[]:m.find.matches(a,m.grep(b,function(a){return 1===a.nodeType}))},m.fn.extend({find:function(a){var b,c=[],d=this,e=d.length;if("string"!=typeof a)return this.pushStack(m(a).filter(function(){for(b=0;e>b;b++)if(m.contains(d[b],this))return!0}));for(b=0;e>b;b++)m.find(a,d[b],c);return c=this.pushStack(e>1?m.unique(c):c),c.selector=this.selector?this.selector+" "+a:a,c},filter:function(a){return this.pushStack(w(this,a||[],!1))},not:function(a){return this.pushStack(w(this,a||[],!0))},is:function(a){return!!w(this,"string"==typeof a&&t.test(a)?m(a):a||[],!1).length}});var x,y=a.document,z=/^(?:\s*(<[\w\W]+>)[^>]*|#([\w-]*))$/,A=m.fn.init=function(a,b){var c,d;if(!a)return this;if("string"==typeof a){if(c="<"===a.charAt(0)&&">"===a.charAt(a.length-1)&&a.length>=3?[null,a,null]:z.exec(a),!c||!c[1]&&b)return!b||b.jquery?(b||x).find(a):this.constructor(b).find(a);if(c[1]){if(b=b instanceof m?b[0]:b,m.merge(this,m.parseHTML(c[1],b&&b.nodeType?b.ownerDocument||b:y,!0)),u.test(c[1])&&m.isPlainObject(b))for(c in b)m.isFunction(this[c])?this[c](b[c]):this.attr(c,b[c]);return this}if(d=y.getElementById(c[2]),d&&d.parentNode){if(d.id!==c[2])return x.find(a);this.length=1,this[0]=d}return this.context=y,this.selector=a,this}return a.nodeType?(this.context=this[0]=a,this.length=1,this):m.isFunction(a)?"undefined"!=typeof x.ready?x.ready(a):a(m):(void 0!==a.selector&&(this.selector=a.selector,this.context=a.context),m.makeArray(a,this))};A.prototype=m.fn,x=m(y);var B=/^(?:parents|prev(?:Until|All))/,C={children:!0,contents:!0,next:!0,prev:!0};m.extend({dir:function(a,b,c){var d=[],e=a[b];while(e&&9!==e.nodeType&&(void 0===c||1!==e.nodeType||!m(e).is(c)))1===e.nodeType&&d.push(e),e=e[b];return d},sibling:function(a,b){for(var c=[];a;a=a.nextSibling)1===a.nodeType&&a!==b&&c.push(a);return c}}),m.fn.extend({has:function(a){var b,c=m(a,this),d=c.length;return this.filter(function(){for(b=0;d>b;b++)if(m.contains(this,c[b]))return!0})},closest:function(a,b){for(var c,d=0,e=this.length,f=[],g=t.test(a)||"string"!=typeof a?m(a,b||this.context):0;e>d;d++)for(c=this[d];c&&c!==b;c=c.parentNode)if(c.nodeType<11&&(g?g.index(c)>-1:1===c.nodeType&&m.find.matchesSelector(c,a))){f.push(c);break}return this.pushStack(f.length>1?m.unique(f):f)},index:function(a){return a?"string"==typeof a?m.inArray(this[0],m(a)):m.inArray(a.jquery?a[0]:a,this):this[0]&&this[0].parentNode?this.first().prevAll().length:-1},add:function(a,b){return this.pushStack(m.unique(m.merge(this.get(),m(a,b))))},addBack:function(a){return this.add(null==a?this.prevObject:this.prevObject.filter(a))}});function D(a,b){do a=a[b];while(a&&1!==a.nodeType);return a}m.each({parent:function(a){var b=a.parentNode;return b&&11!==b.nodeType?b:null},parents:function(a){return m.dir(a,"parentNode")},parentsUntil:function(a,b,c){return m.dir(a,"parentNode",c)},next:function(a){return D(a,"nextSibling")},prev:function(a){return D(a,"previousSibling")},nextAll:function(a){return m.dir(a,"nextSibling")},prevAll:function(a){return m.dir(a,"previousSibling")},nextUntil:function(a,b,c){return m.dir(a,"nextSibling",c)},prevUntil:function(a,b,c){return m.dir(a,"previousSibling",c)},siblings:function(a){return m.sibling((a.parentNode||{}).firstChild,a)},children:function(a){return m.sibling(a.firstChild)},contents:function(a){return m.nodeName(a,"iframe")?a.contentDocument||a.contentWindow.document:m.merge([],a.childNodes)}},function(a,b){m.fn[a]=function(c,d){var e=m.map(this,b,c);return"Until"!==a.slice(-5)&&(d=c),d&&"string"==typeof d&&(e=m.filter(d,e)),this.length>1&&(C[a]||(e=m.unique(e)),B.test(a)&&(e=e.reverse())),this.pushStack(e)}});var E=/\S+/g,F={};function G(a){var b=F[a]={};return m.each(a.match(E)||[],function(a,c){b[c]=!0}),b}m.Callbacks=function(a){a="string"==typeof a?F[a]||G(a):m.extend({},a);var b,c,d,e,f,g,h=[],i=!a.once&&[],j=function(l){for(c=a.memory&&l,d=!0,f=g||0,g=0,e=h.length,b=!0;h&&e>f;f++)if(h[f].apply(l[0],l[1])===!1&&a.stopOnFalse){c=!1;break}b=!1,h&&(i?i.length&&j(i.shift()):c?h=[]:k.disable())},k={add:function(){if(h){var d=h.length;!function f(b){m.each(b,function(b,c){var d=m.type(c);"function"===d?a.unique&&k.has(c)||h.push(c):c&&c.length&&"string"!==d&&f(c)})}(arguments),b?e=h.length:c&&(g=d,j(c))}return this},remove:function(){return h&&m.each(arguments,function(a,c){var d;while((d=m.inArray(c,h,d))>-1)h.splice(d,1),b&&(e>=d&&e--,f>=d&&f--)}),this},has:function(a){return a?m.inArray(a,h)>-1:!(!h||!h.length)},empty:function(){return h=[],e=0,this},disable:function(){return h=i=c=void 0,this},disabled:function(){return!h},lock:function(){return i=void 0,c||k.disable(),this},locked:function(){return!i},fireWith:function(a,c){return!h||d&&!i||(c=c||[],c=[a,c.slice?c.slice():c],b?i.push(c):j(c)),this},fire:function(){return k.fireWith(this,arguments),this},fired:function(){return!!d}};return k},m.extend({Deferred:function(a){var b=[["resolve","done",m.Callbacks("once memory"),"resolved"],["reject","fail",m.Callbacks("once memory"),"rejected"],["notify","progress",m.Callbacks("memory")]],c="pending",d={state:function(){return c},always:function(){return e.done(arguments).fail(arguments),this},then:function(){var a=arguments;return m.Deferred(function(c){m.each(b,function(b,f){var g=m.isFunction(a[b])&&a[b];e[f[1]](function(){var a=g&&g.apply(this,arguments);a&&m.isFunction(a.promise)?a.promise().done(c.resolve).fail(c.reject).progress(c.notify):c[f[0]+"With"](this===d?c.promise():this,g?[a]:arguments)})}),a=null}).promise()},promise:function(a){return null!=a?m.extend(a,d):d}},e={};return d.pipe=d.then,m.each(b,function(a,f){var g=f[2],h=f[3];d[f[1]]=g.add,h&&g.add(function(){c=h},b[1^a][2].disable,b[2][2].lock),e[f[0]]=function(){return e[f[0]+"With"](this===e?d:this,arguments),this},e[f[0]+"With"]=g.fireWith}),d.promise(e),a&&a.call(e,e),e},when:function(a){var b=0,c=d.call(arguments),e=c.length,f=1!==e||a&&m.isFunction(a.promise)?e:0,g=1===f?a:m.Deferred(),h=function(a,b,c){return function(e){b[a]=this,c[a]=arguments.length>1?d.call(arguments):e,c===i?g.notifyWith(b,c):--f||g.resolveWith(b,c)}},i,j,k;if(e>1)for(i=new Array(e),j=new Array(e),k=new Array(e);e>b;b++)c[b]&&m.isFunction(c[b].promise)?c[b].promise().done(h(b,k,c)).fail(g.reject).progress(h(b,j,i)):--f;return f||g.resolveWith(k,c),g.promise()}});var H;m.fn.ready=function(a){return m.ready.promise().done(a),this},m.extend({isReady:!1,readyWait:1,holdReady:function(a){a?m.readyWait++:m.ready(!0)},ready:function(a){if(a===!0?!--m.readyWait:!m.isReady){if(!y.body)return setTimeout(m.ready);m.isReady=!0,a!==!0&&--m.readyWait>0||(H.resolveWith(y,[m]),m.fn.triggerHandler&&(m(y).triggerHandler("ready"),m(y).off("ready")))}}});function I(){y.addEventListener?(y.removeEventListener("DOMContentLoaded",J,!1),a.removeEventListener("load",J,!1)):(y.detachEvent("onreadystatechange",J),a.detachEvent("onload",J))}function J(){(y.addEventListener||"load"===event.type||"complete"===y.readyState)&&(I(),m.ready())}m.ready.promise=function(b){if(!H)if(H=m.Deferred(),"complete"===y.readyState)setTimeout(m.ready);else if(y.addEventListener)y.addEventListener("DOMContentLoaded",J,!1),a.addEventListener("load",J,!1);else{y.attachEvent("onreadystatechange",J),a.attachEvent("onload",J);var c=!1;try{c=null==a.frameElement&&y.documentElement}catch(d){}c&&c.doScroll&&!function e(){if(!m.isReady){try{c.doScroll("left")}catch(a){return setTimeout(e,50)}I(),m.ready()}}()}return H.promise(b)};var K="undefined",L;for(L in m(k))break;k.ownLast="0"!==L,k.inlineBlockNeedsLayout=!1,m(function(){var a,b,c,d;c=y.getElementsByTagName("body")[0],c&&c.style&&(b=y.createElement("div"),d=y.createElement("div"),d.style.cssText="position:absolute;border:0;width:0;height:0;top:0;left:-9999px",c.appendChild(d).appendChild(b),typeof b.style.zoom!==K&&(b.style.cssText="display:inline;margin:0;border:0;padding:1px;width:1px;zoom:1",k.inlineBlockNeedsLayout=a=3===b.offsetWidth,a&&(c.style.zoom=1)),c.removeChild(d))}),function(){var a=y.createElement("div");if(null==k.deleteExpando){k.deleteExpando=!0;try{delete a.test}catch(b){k.deleteExpando=!1}}a=null}(),m.acceptData=function(a){var b=m.noData[(a.nodeName+" ").toLowerCase()],c=+a.nodeType||1;return 1!==c&&9!==c?!1:!b||b!==!0&&a.getAttribute("classid")===b};var M=/^(?:\{[\w\W]*\}|\[[\w\W]*\])$/,N=/([A-Z])/g;function O(a,b,c){if(void 0===c&&1===a.nodeType){var d="data-"+b.replace(N,"-$1").toLowerCase();if(c=a.getAttribute(d),"string"==typeof c){try{c="true"===c?!0:"false"===c?!1:"null"===c?null:+c+""===c?+c:M.test(c)?m.parseJSON(c):c}catch(e){}m.data(a,b,c)}else c=void 0}return c}function P(a){var b;for(b in a)if(("data"!==b||!m.isEmptyObject(a[b]))&&"toJSON"!==b)return!1;

return!0}function Q(a,b,d,e){if(m.acceptData(a)){var f,g,h=m.expando,i=a.nodeType,j=i?m.cache:a,k=i?a[h]:a[h]&&h;if(k&&j[k]&&(e||j[k].data)||void 0!==d||"string"!=typeof b)return k||(k=i?a[h]=c.pop()||m.guid++:h),j[k]||(j[k]=i?{}:{toJSON:m.noop}),("object"==typeof b||"function"==typeof b)&&(e?j[k]=m.extend(j[k],b):j[k].data=m.extend(j[k].data,b)),g=j[k],e||(g.data||(g.data={}),g=g.data),void 0!==d&&(g[m.camelCase(b)]=d),"string"==typeof b?(f=g[b],null==f&&(f=g[m.camelCase(b)])):f=g,f}}function R(a,b,c){if(m.acceptData(a)){var d,e,f=a.nodeType,g=f?m.cache:a,h=f?a[m.expando]:m.expando;if(g[h]){if(b&&(d=c?g[h]:g[h].data)){m.isArray(b)?b=b.concat(m.map(b,m.camelCase)):b in d?b=[b]:(b=m.camelCase(b),b=b in d?[b]:b.split(" ")),e=b.length;while(e--)delete d[b[e]];if(c?!P(d):!m.isEmptyObject(d))return}(c||(delete g[h].data,P(g[h])))&&(f?m.cleanData([a],!0):k.deleteExpando||g!=g.window?delete g[h]:g[h]=null)}}}m.extend({cache:{},noData:{"applet ":!0,"embed ":!0,"object ":"clsid:D27CDB6E-AE6D-11cf-96B8-444553540000"},hasData:function(a){return a=a.nodeType?m.cache[a[m.expando]]:a[m.expando],!!a&&!P(a)},data:function(a,b,c){return Q(a,b,c)},removeData:function(a,b){return R(a,b)},_data:function(a,b,c){return Q(a,b,c,!0)},_removeData:function(a,b){return R(a,b,!0)}}),m.fn.extend({data:function(a,b){var c,d,e,f=this[0],g=f&&f.attributes;if(void 0===a){if(this.length&&(e=m.data(f),1===f.nodeType&&!m._data(f,"parsedAttrs"))){c=g.length;while(c--)g[c]&&(d=g[c].name,0===d.indexOf("data-")&&(d=m.camelCase(d.slice(5)),O(f,d,e[d])));m._data(f,"parsedAttrs",!0)}return e}return"object"==typeof a?this.each(function(){m.data(this,a)}):arguments.length>1?this.each(function(){m.data(this,a,b)}):f?O(f,a,m.data(f,a)):void 0},removeData:function(a){return this.each(function(){m.removeData(this,a)})}}),m.extend({queue:function(a,b,c){var d;return a?(b=(b||"fx")+"queue",d=m._data(a,b),c&&(!d||m.isArray(c)?d=m._data(a,b,m.makeArray(c)):d.push(c)),d||[]):void 0},dequeue:function(a,b){b=b||"fx";var c=m.queue(a,b),d=c.length,e=c.shift(),f=m._queueHooks(a,b),g=function(){m.dequeue(a,b)};"inprogress"===e&&(e=c.shift(),d--),e&&("fx"===b&&c.unshift("inprogress"),delete f.stop,e.call(a,g,f)),!d&&f&&f.empty.fire()},_queueHooks:function(a,b){var c=b+"queueHooks";return m._data(a,c)||m._data(a,c,{empty:m.Callbacks("once memory").add(function(){m._removeData(a,b+"queue"),m._removeData(a,c)})})}}),m.fn.extend({queue:function(a,b){var c=2;return"string"!=typeof a&&(b=a,a="fx",c--),arguments.length<c?m.queue(this[0],a):void 0===b?this:this.each(function(){var c=m.queue(this,a,b);m._queueHooks(this,a),"fx"===a&&"inprogress"!==c[0]&&m.dequeue(this,a)})},dequeue:function(a){return this.each(function(){m.dequeue(this,a)})},clearQueue:function(a){return this.queue(a||"fx",[])},promise:function(a,b){var c,d=1,e=m.Deferred(),f=this,g=this.length,h=function(){--d||e.resolveWith(f,[f])};"string"!=typeof a&&(b=a,a=void 0),a=a||"fx";while(g--)c=m._data(f[g],a+"queueHooks"),c&&c.empty&&(d++,c.empty.add(h));return h(),e.promise(b)}});var S=/[+-]?(?:\d*\.|)\d+(?:[eE][+-]?\d+|)/.source,T=["Top","Right","Bottom","Left"],U=function(a,b){return a=b||a,"none"===m.css(a,"display")||!m.contains(a.ownerDocument,a)},V=m.access=function(a,b,c,d,e,f,g){var h=0,i=a.length,j=null==c;if("object"===m.type(c)){e=!0;for(h in c)m.access(a,b,h,c[h],!0,f,g)}else if(void 0!==d&&(e=!0,m.isFunction(d)||(g=!0),j&&(g?(b.call(a,d),b=null):(j=b,b=function(a,b,c){return j.call(m(a),c)})),b))for(;i>h;h++)b(a[h],c,g?d:d.call(a[h],h,b(a[h],c)));return e?a:j?b.call(a):i?b(a[0],c):f},W=/^(?:checkbox|radio)$/i;!function(){var a=y.createElement("input"),b=y.createElement("div"),c=y.createDocumentFragment();if(b.innerHTML="  <link/><table></table><a href='/a'>a</a><input type='checkbox'/>",k.leadingWhitespace=3===b.firstChild.nodeType,k.tbody=!b.getElementsByTagName("tbody").length,k.htmlSerialize=!!b.getElementsByTagName("link").length,k.html5Clone="<:nav></:nav>"!==y.createElement("nav").cloneNode(!0).outerHTML,a.type="checkbox",a.checked=!0,c.appendChild(a),k.appendChecked=a.checked,b.innerHTML="<textarea>x</textarea>",k.noCloneChecked=!!b.cloneNode(!0).lastChild.defaultValue,c.appendChild(b),b.innerHTML="<input type='radio' checked='checked' name='t'/>",k.checkClone=b.cloneNode(!0).cloneNode(!0).lastChild.checked,k.noCloneEvent=!0,b.attachEvent&&(b.attachEvent("onclick",function(){k.noCloneEvent=!1}),b.cloneNode(!0).click()),null==k.deleteExpando){k.deleteExpando=!0;try{delete b.test}catch(d){k.deleteExpando=!1}}}(),function(){var b,c,d=y.createElement("div");for(b in{submit:!0,change:!0,focusin:!0})c="on"+b,(k[b+"Bubbles"]=c in a)||(d.setAttribute(c,"t"),k[b+"Bubbles"]=d.attributes[c].expando===!1);d=null}();var X=/^(?:input|select|textarea)$/i,Y=/^key/,Z=/^(?:mouse|pointer|contextmenu)|click/,$=/^(?:focusinfocus|focusoutblur)$/,_=/^([^.]*)(?:\.(.+)|)$/;function aa(){return!0}function ba(){return!1}function ca(){try{return y.activeElement}catch(a){}}m.event={global:{},add:function(a,b,c,d,e){var f,g,h,i,j,k,l,n,o,p,q,r=m._data(a);if(r){c.handler&&(i=c,c=i.handler,e=i.selector),c.guid||(c.guid=m.guid++),(g=r.events)||(g=r.events={}),(k=r.handle)||(k=r.handle=function(a){return typeof m===K||a&&m.event.triggered===a.type?void 0:m.event.dispatch.apply(k.elem,arguments)},k.elem=a),b=(b||"").match(E)||[""],h=b.length;while(h--)f=_.exec(b[h])||[],o=q=f[1],p=(f[2]||"").split(".").sort(),o&&(j=m.event.special[o]||{},o=(e?j.delegateType:j.bindType)||o,j=m.event.special[o]||{},l=m.extend({type:o,origType:q,data:d,handler:c,guid:c.guid,selector:e,needsContext:e&&m.expr.match.needsContext.test(e),namespace:p.join(".")},i),(n=g[o])||(n=g[o]=[],n.delegateCount=0,j.setup&&j.setup.call(a,d,p,k)!==!1||(a.addEventListener?a.addEventListener(o,k,!1):a.attachEvent&&a.attachEvent("on"+o,k))),j.add&&(j.add.call(a,l),l.handler.guid||(l.handler.guid=c.guid)),e?n.splice(n.delegateCount++,0,l):n.push(l),m.event.global[o]=!0);a=null}},remove:function(a,b,c,d,e){var f,g,h,i,j,k,l,n,o,p,q,r=m.hasData(a)&&m._data(a);if(r&&(k=r.events)){b=(b||"").match(E)||[""],j=b.length;while(j--)if(h=_.exec(b[j])||[],o=q=h[1],p=(h[2]||"").split(".").sort(),o){l=m.event.special[o]||{},o=(d?l.delegateType:l.bindType)||o,n=k[o]||[],h=h[2]&&new RegExp("(^|\\.)"+p.join("\\.(?:.*\\.|)")+"(\\.|$)"),i=f=n.length;while(f--)g=n[f],!e&&q!==g.origType||c&&c.guid!==g.guid||h&&!h.test(g.namespace)||d&&d!==g.selector&&("**"!==d||!g.selector)||(n.splice(f,1),g.selector&&n.delegateCount--,l.remove&&l.remove.call(a,g));i&&!n.length&&(l.teardown&&l.teardown.call(a,p,r.handle)!==!1||m.removeEvent(a,o,r.handle),delete k[o])}else for(o in k)m.event.remove(a,o+b[j],c,d,!0);m.isEmptyObject(k)&&(delete r.handle,m._removeData(a,"events"))}},trigger:function(b,c,d,e){var f,g,h,i,k,l,n,o=[d||y],p=j.call(b,"type")?b.type:b,q=j.call(b,"namespace")?b.namespace.split("."):[];if(h=l=d=d||y,3!==d.nodeType&&8!==d.nodeType&&!$.test(p+m.event.triggered)&&(p.indexOf(".")>=0&&(q=p.split("."),p=q.shift(),q.sort()),g=p.indexOf(":")<0&&"on"+p,b=b[m.expando]?b:new m.Event(p,"object"==typeof b&&b),b.isTrigger=e?2:3,b.namespace=q.join("."),b.namespace_re=b.namespace?new RegExp("(^|\\.)"+q.join("\\.(?:.*\\.|)")+"(\\.|$)"):null,b.result=void 0,b.target||(b.target=d),c=null==c?[b]:m.makeArray(c,[b]),k=m.event.special[p]||{},e||!k.trigger||k.trigger.apply(d,c)!==!1)){if(!e&&!k.noBubble&&!m.isWindow(d)){for(i=k.delegateType||p,$.test(i+p)||(h=h.parentNode);h;h=h.parentNode)o.push(h),l=h;l===(d.ownerDocument||y)&&o.push(l.defaultView||l.parentWindow||a)}n=0;while((h=o[n++])&&!b.isPropagationStopped())b.type=n>1?i:k.bindType||p,f=(m._data(h,"events")||{})[b.type]&&m._data(h,"handle"),f&&f.apply(h,c),f=g&&h[g],f&&f.apply&&m.acceptData(h)&&(b.result=f.apply(h,c),b.result===!1&&b.preventDefault());if(b.type=p,!e&&!b.isDefaultPrevented()&&(!k._default||k._default.apply(o.pop(),c)===!1)&&m.acceptData(d)&&g&&d[p]&&!m.isWindow(d)){l=d[g],l&&(d[g]=null),m.event.triggered=p;try{d[p]()}catch(r){}m.event.triggered=void 0,l&&(d[g]=l)}return b.result}},dispatch:function(a){a=m.event.fix(a);var b,c,e,f,g,h=[],i=d.call(arguments),j=(m._data(this,"events")||{})[a.type]||[],k=m.event.special[a.type]||{};if(i[0]=a,a.delegateTarget=this,!k.preDispatch||k.preDispatch.call(this,a)!==!1){h=m.event.handlers.call(this,a,j),b=0;while((f=h[b++])&&!a.isPropagationStopped()){a.currentTarget=f.elem,g=0;while((e=f.handlers[g++])&&!a.isImmediatePropagationStopped())(!a.namespace_re||a.namespace_re.test(e.namespace))&&(a.handleObj=e,a.data=e.data,c=((m.event.special[e.origType]||{}).handle||e.handler).apply(f.elem,i),void 0!==c&&(a.result=c)===!1&&(a.preventDefault(),a.stopPropagation()))}return k.postDispatch&&k.postDispatch.call(this,a),a.result}},handlers:function(a,b){var c,d,e,f,g=[],h=b.delegateCount,i=a.target;if(h&&i.nodeType&&(!a.button||"click"!==a.type))for(;i!=this;i=i.parentNode||this)if(1===i.nodeType&&(i.disabled!==!0||"click"!==a.type)){for(e=[],f=0;h>f;f++)d=b[f],c=d.selector+" ",void 0===e[c]&&(e[c]=d.needsContext?m(c,this).index(i)>=0:m.find(c,this,null,[i]).length),e[c]&&e.push(d);e.length&&g.push({elem:i,handlers:e})}return h<b.length&&g.push({elem:this,handlers:b.slice(h)}),g},fix:function(a){if(a[m.expando])return a;var b,c,d,e=a.type,f=a,g=this.fixHooks[e];g||(this.fixHooks[e]=g=Z.test(e)?this.mouseHooks:Y.test(e)?this.keyHooks:{}),d=g.props?this.props.concat(g.props):this.props,a=new m.Event(f),b=d.length;while(b--)c=d[b],a[c]=f[c];return a.target||(a.target=f.srcElement||y),3===a.target.nodeType&&(a.target=a.target.parentNode),a.metaKey=!!a.metaKey,g.filter?g.filter(a,f):a},props:"altKey bubbles cancelable ctrlKey currentTarget eventPhase metaKey relatedTarget shiftKey target timeStamp view which".split(" "),fixHooks:{},keyHooks:{props:"char charCode key keyCode".split(" "),filter:function(a,b){return null==a.which&&(a.which=null!=b.charCode?b.charCode:b.keyCode),a}},mouseHooks:{props:"button buttons clientX clientY fromElement offsetX offsetY pageX pageY screenX screenY toElement".split(" "),filter:function(a,b){var c,d,e,f=b.button,g=b.fromElement;return null==a.pageX&&null!=b.clientX&&(d=a.target.ownerDocument||y,e=d.documentElement,c=d.body,a.pageX=b.clientX+(e&&e.scrollLeft||c&&c.scrollLeft||0)-(e&&e.clientLeft||c&&c.clientLeft||0),a.pageY=b.clientY+(e&&e.scrollTop||c&&c.scrollTop||0)-(e&&e.clientTop||c&&c.clientTop||0)),!a.relatedTarget&&g&&(a.relatedTarget=g===a.target?b.toElement:g),a.which||void 0===f||(a.which=1&f?1:2&f?3:4&f?2:0),a}},special:{load:{noBubble:!0},focus:{trigger:function(){if(this!==ca()&&this.focus)try{return this.focus(),!1}catch(a){}},delegateType:"focusin"},blur:{trigger:function(){return this===ca()&&this.blur?(this.blur(),!1):void 0},delegateType:"focusout"},click:{trigger:function(){return m.nodeName(this,"input")&&"checkbox"===this.type&&this.click?(this.click(),!1):void 0},_default:function(a){return m.nodeName(a.target,"a")}},beforeunload:{postDispatch:function(a){void 0!==a.result&&a.originalEvent&&(a.originalEvent.returnValue=a.result)}}},simulate:function(a,b,c,d){var e=m.extend(new m.Event,c,{type:a,isSimulated:!0,originalEvent:{}});d?m.event.trigger(e,null,b):m.event.dispatch.call(b,e),e.isDefaultPrevented()&&c.preventDefault()}},m.removeEvent=y.removeEventListener?function(a,b,c){a.removeEventListener&&a.removeEventListener(b,c,!1)}:function(a,b,c){var d="on"+b;a.detachEvent&&(typeof a[d]===K&&(a[d]=null),a.detachEvent(d,c))},m.Event=function(a,b){return this instanceof m.Event?(a&&a.type?(this.originalEvent=a,this.type=a.type,this.isDefaultPrevented=a.defaultPrevented||void 0===a.defaultPrevented&&a.returnValue===!1?aa:ba):this.type=a,b&&m.extend(this,b),this.timeStamp=a&&a.timeStamp||m.now(),void(this[m.expando]=!0)):new m.Event(a,b)},m.Event.prototype={isDefaultPrevented:ba,isPropagationStopped:ba,isImmediatePropagationStopped:ba,preventDefault:function(){var a=this.originalEvent;this.isDefaultPrevented=aa,a&&(a.preventDefault?a.preventDefault():a.returnValue=!1)},stopPropagation:function(){var a=this.originalEvent;this.isPropagationStopped=aa,a&&(a.stopPropagation&&a.stopPropagation(),a.cancelBubble=!0)},stopImmediatePropagation:function(){var a=this.originalEvent;this.isImmediatePropagationStopped=aa,a&&a.stopImmediatePropagation&&a.stopImmediatePropagation(),this.stopPropagation()}},m.each({mouseenter:"mouseover",mouseleave:"mouseout",pointerenter:"pointerover",pointerleave:"pointerout"},function(a,b){m.event.special[a]={delegateType:b,bindType:b,handle:function(a){var c,d=this,e=a.relatedTarget,f=a.handleObj;return(!e||e!==d&&!m.contains(d,e))&&(a.type=f.origType,c=f.handler.apply(this,arguments),a.type=b),c}}}),k.submitBubbles||(m.event.special.submit={setup:function(){return m.nodeName(this,"form")?!1:void m.event.add(this,"click._submit keypress._submit",function(a){var b=a.target,c=m.nodeName(b,"input")||m.nodeName(b,"button")?b.form:void 0;c&&!m._data(c,"submitBubbles")&&(m.event.add(c,"submit._submit",function(a){a._submit_bubble=!0}),m._data(c,"submitBubbles",!0))})},postDispatch:function(a){a._submit_bubble&&(delete a._submit_bubble,this.parentNode&&!a.isTrigger&&m.event.simulate("submit",this.parentNode,a,!0))},teardown:function(){return m.nodeName(this,"form")?!1:void m.event.remove(this,"._submit")}}),k.changeBubbles||(m.event.special.change={setup:function(){return X.test(this.nodeName)?(("checkbox"===this.type||"radio"===this.type)&&(m.event.add(this,"propertychange._change",function(a){"checked"===a.originalEvent.propertyName&&(this._just_changed=!0)}),m.event.add(this,"click._change",function(a){this._just_changed&&!a.isTrigger&&(this._just_changed=!1),m.event.simulate("change",this,a,!0)})),!1):void m.event.add(this,"beforeactivate._change",function(a){var b=a.target;X.test(b.nodeName)&&!m._data(b,"changeBubbles")&&(m.event.add(b,"change._change",function(a){!this.parentNode||a.isSimulated||a.isTrigger||m.event.simulate("change",this.parentNode,a,!0)}),m._data(b,"changeBubbles",!0))})},handle:function(a){var b=a.target;return this!==b||a.isSimulated||a.isTrigger||"radio"!==b.type&&"checkbox"!==b.type?a.handleObj.handler.apply(this,arguments):void 0},teardown:function(){return m.event.remove(this,"._change"),!X.test(this.nodeName)}}),k.focusinBubbles||m.each({focus:"focusin",blur:"focusout"},function(a,b){var c=function(a){m.event.simulate(b,a.target,m.event.fix(a),!0)};m.event.special[b]={setup:function(){var d=this.ownerDocument||this,e=m._data(d,b);e||d.addEventListener(a,c,!0),m._data(d,b,(e||0)+1)},teardown:function(){var d=this.ownerDocument||this,e=m._data(d,b)-1;e?m._data(d,b,e):(d.removeEventListener(a,c,!0),m._removeData(d,b))}}}),m.fn.extend({on:function(a,b,c,d,e){var f,g;if("object"==typeof a){"string"!=typeof b&&(c=c||b,b=void 0);for(f in a)this.on(f,b,c,a[f],e);return this}if(null==c&&null==d?(d=b,c=b=void 0):null==d&&("string"==typeof b?(d=c,c=void 0):(d=c,c=b,b=void 0)),d===!1)d=ba;else if(!d)return this;return 1===e&&(g=d,d=function(a){return m().off(a),g.apply(this,arguments)},d.guid=g.guid||(g.guid=m.guid++)),this.each(function(){m.event.add(this,a,d,c,b)})},one:function(a,b,c,d){return this.on(a,b,c,d,1)},off:function(a,b,c){var d,e;if(a&&a.preventDefault&&a.handleObj)return d=a.handleObj,m(a.delegateTarget).off(d.namespace?d.origType+"."+d.namespace:d.origType,d.selector,d.handler),this;if("object"==typeof a){for(e in a)this.off(e,b,a[e]);return this}return(b===!1||"function"==typeof b)&&(c=b,b=void 0),c===!1&&(c=ba),this.each(function(){m.event.remove(this,a,c,b)})},trigger:function(a,b){return this.each(function(){m.event.trigger(a,b,this)})},triggerHandler:function(a,b){var c=this[0];return c?m.event.trigger(a,b,c,!0):void 0}});function da(a){var b=ea.split("|"),c=a.createDocumentFragment();if(c.createElement)while(b.length)c.createElement(b.pop());return c}var ea="abbr|article|aside|audio|bdi|canvas|data|datalist|details|figcaption|figure|footer|header|hgroup|mark|meter|nav|output|progress|section|summary|time|video",fa=/ jQuery\d+="(?:null|\d+)"/g,ga=new RegExp("<(?:"+ea+")[\\s/>]","i"),ha=/^\s+/,ia=/<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/gi,ja=/<([\w:]+)/,ka=/<tbody/i,la=/<|&#?\w+;/,ma=/<(?:script|style|link)/i,na=/checked\s*(?:[^=]|=\s*.checked.)/i,oa=/^$|\/(?:java|ecma)script/i,pa=/^true\/(.*)/,qa=/^\s*<!(?:\[CDATA\[|--)|(?:\]\]|--)>\s*$/g,ra={option:[1,"<select multiple='multiple'>","</select>"],legend:[1,"<fieldset>","</fieldset>"],area:[1,"<map>","</map>"],param:[1,"<object>","</object>"],thead:[1,"<table>","</table>"],tr:[2,"<table><tbody>","</tbody></table>"],col:[2,"<table><tbody></tbody><colgroup>","</colgroup></table>"],td:[3,"<table><tbody><tr>","</tr></tbody></table>"],_default:k.htmlSerialize?[0,"",""]:[1,"X<div>","</div>"]},sa=da(y),ta=sa.appendChild(y.createElement("div"));ra.optgroup=ra.option,ra.tbody=ra.tfoot=ra.colgroup=ra.caption=ra.thead,ra.th=ra.td;function ua(a,b){var c,d,e=0,f=typeof a.getElementsByTagName!==K?a.getElementsByTagName(b||"*"):typeof a.querySelectorAll!==K?a.querySelectorAll(b||"*"):void 0;if(!f)for(f=[],c=a.childNodes||a;null!=(d=c[e]);e++)!b||m.nodeName(d,b)?f.push(d):m.merge(f,ua(d,b));return void 0===b||b&&m.nodeName(a,b)?m.merge([a],f):f}function va(a){W.test(a.type)&&(a.defaultChecked=a.checked)}function wa(a,b){return m.nodeName(a,"table")&&m.nodeName(11!==b.nodeType?b:b.firstChild,"tr")?a.getElementsByTagName("tbody")[0]||a.appendChild(a.ownerDocument.createElement("tbody")):a}function xa(a){return a.type=(null!==m.find.attr(a,"type"))+"/"+a.type,a}function ya(a){var b=pa.exec(a.type);return b?a.type=b[1]:a.removeAttribute("type"),a}function za(a,b){for(var c,d=0;null!=(c=a[d]);d++)m._data(c,"globalEval",!b||m._data(b[d],"globalEval"))}function Aa(a,b){if(1===b.nodeType&&m.hasData(a)){var c,d,e,f=m._data(a),g=m._data(b,f),h=f.events;if(h){delete g.handle,g.events={};for(c in h)for(d=0,e=h[c].length;e>d;d++)m.event.add(b,c,h[c][d])}g.data&&(g.data=m.extend({},g.data))}}function Ba(a,b){var c,d,e;if(1===b.nodeType){if(c=b.nodeName.toLowerCase(),!k.noCloneEvent&&b[m.expando]){e=m._data(b);for(d in e.events)m.removeEvent(b,d,e.handle);b.removeAttribute(m.expando)}"script"===c&&b.text!==a.text?(xa(b).text=a.text,ya(b)):"object"===c?(b.parentNode&&(b.outerHTML=a.outerHTML),k.html5Clone&&a.innerHTML&&!m.trim(b.innerHTML)&&(b.innerHTML=a.innerHTML)):"input"===c&&W.test(a.type)?(b.defaultChecked=b.checked=a.checked,b.value!==a.value&&(b.value=a.value)):"option"===c?b.defaultSelected=b.selected=a.defaultSelected:("input"===c||"textarea"===c)&&(b.defaultValue=a.defaultValue)}}m.extend({clone:function(a,b,c){var d,e,f,g,h,i=m.contains(a.ownerDocument,a);if(k.html5Clone||m.isXMLDoc(a)||!ga.test("<"+a.nodeName+">")?f=a.cloneNode(!0):(ta.innerHTML=a.outerHTML,ta.removeChild(f=ta.firstChild)),!(k.noCloneEvent&&k.noCloneChecked||1!==a.nodeType&&11!==a.nodeType||m.isXMLDoc(a)))for(d=ua(f),h=ua(a),g=0;null!=(e=h[g]);++g)d[g]&&Ba(e,d[g]);if(b)if(c)for(h=h||ua(a),d=d||ua(f),g=0;null!=(e=h[g]);g++)Aa(e,d[g]);else Aa(a,f);return d=ua(f,"script"),d.length>0&&za(d,!i&&ua(a,"script")),d=h=e=null,f},buildFragment:function(a,b,c,d){for(var e,f,g,h,i,j,l,n=a.length,o=da(b),p=[],q=0;n>q;q++)if(f=a[q],f||0===f)if("object"===m.type(f))m.merge(p,f.nodeType?[f]:f);else if(la.test(f)){h=h||o.appendChild(b.createElement("div")),i=(ja.exec(f)||["",""])[1].toLowerCase(),l=ra[i]||ra._default,h.innerHTML=l[1]+f.replace(ia,"<$1></$2>")+l[2],e=l[0];while(e--)h=h.lastChild;if(!k.leadingWhitespace&&ha.test(f)&&p.push(b.createTextNode(ha.exec(f)[0])),!k.tbody){f="table"!==i||ka.test(f)?"<table>"!==l[1]||ka.test(f)?0:h:h.firstChild,e=f&&f.childNodes.length;while(e--)m.nodeName(j=f.childNodes[e],"tbody")&&!j.childNodes.length&&f.removeChild(j)}m.merge(p,h.childNodes),h.textContent="";while(h.firstChild)h.removeChild(h.firstChild);h=o.lastChild}else p.push(b.createTextNode(f));h&&o.removeChild(h),k.appendChecked||m.grep(ua(p,"input"),va),q=0;while(f=p[q++])if((!d||-1===m.inArray(f,d))&&(g=m.contains(f.ownerDocument,f),h=ua(o.appendChild(f),"script"),g&&za(h),c)){e=0;while(f=h[e++])oa.test(f.type||"")&&c.push(f)}return h=null,o},cleanData:function(a,b){for(var d,e,f,g,h=0,i=m.expando,j=m.cache,l=k.deleteExpando,n=m.event.special;null!=(d=a[h]);h++)if((b||m.acceptData(d))&&(f=d[i],g=f&&j[f])){if(g.events)for(e in g.events)n[e]?m.event.remove(d,e):m.removeEvent(d,e,g.handle);j[f]&&(delete j[f],l?delete d[i]:typeof d.removeAttribute!==K?d.removeAttribute(i):d[i]=null,c.push(f))}}}),m.fn.extend({text:function(a){return V(this,function(a){return void 0===a?m.text(this):this.empty().append((this[0]&&this[0].ownerDocument||y).createTextNode(a))},null,a,arguments.length)},append:function(){return this.domManip(arguments,function(a){if(1===this.nodeType||11===this.nodeType||9===this.nodeType){var b=wa(this,a);b.appendChild(a)}})},prepend:function(){return this.domManip(arguments,function(a){if(1===this.nodeType||11===this.nodeType||9===this.nodeType){var b=wa(this,a);b.insertBefore(a,b.firstChild)}})},before:function(){return this.domManip(arguments,function(a){this.parentNode&&this.parentNode.insertBefore(a,this)})},after:function(){return this.domManip(arguments,function(a){this.parentNode&&this.parentNode.insertBefore(a,this.nextSibling)})},remove:function(a,b){for(var c,d=a?m.filter(a,this):this,e=0;null!=(c=d[e]);e++)b||1!==c.nodeType||m.cleanData(ua(c)),c.parentNode&&(b&&m.contains(c.ownerDocument,c)&&za(ua(c,"script")),c.parentNode.removeChild(c));return this},empty:function(){for(var a,b=0;null!=(a=this[b]);b++){1===a.nodeType&&m.cleanData(ua(a,!1));while(a.firstChild)a.removeChild(a.firstChild);a.options&&m.nodeName(a,"select")&&(a.options.length=0)}return this},clone:function(a,b){return a=null==a?!1:a,b=null==b?a:b,this.map(function(){return m.clone(this,a,b)})},html:function(a){return V(this,function(a){var b=this[0]||{},c=0,d=this.length;if(void 0===a)return 1===b.nodeType?b.innerHTML.replace(fa,""):void 0;if(!("string"!=typeof a||ma.test(a)||!k.htmlSerialize&&ga.test(a)||!k.leadingWhitespace&&ha.test(a)||ra[(ja.exec(a)||["",""])[1].toLowerCase()])){a=a.replace(ia,"<$1></$2>");try{for(;d>c;c++)b=this[c]||{},1===b.nodeType&&(m.cleanData(ua(b,!1)),b.innerHTML=a);b=0}catch(e){}}b&&this.empty().append(a)},null,a,arguments.length)},replaceWith:function(){var a=arguments[0];return this.domManip(arguments,function(b){a=this.parentNode,m.cleanData(ua(this)),a&&a.replaceChild(b,this)}),a&&(a.length||a.nodeType)?this:this.remove()},detach:function(a){return this.remove(a,!0)},domManip:function(a,b){a=e.apply([],a);var c,d,f,g,h,i,j=0,l=this.length,n=this,o=l-1,p=a[0],q=m.isFunction(p);if(q||l>1&&"string"==typeof p&&!k.checkClone&&na.test(p))return this.each(function(c){var d=n.eq(c);q&&(a[0]=p.call(this,c,d.html())),d.domManip(a,b)});if(l&&(i=m.buildFragment(a,this[0].ownerDocument,!1,this),c=i.firstChild,1===i.childNodes.length&&(i=c),c)){for(g=m.map(ua(i,"script"),xa),f=g.length;l>j;j++)d=i,j!==o&&(d=m.clone(d,!0,!0),f&&m.merge(g,ua(d,"script"))),b.call(this[j],d,j);if(f)for(h=g[g.length-1].ownerDocument,m.map(g,ya),j=0;f>j;j++)d=g[j],oa.test(d.type||"")&&!m._data(d,"globalEval")&&m.contains(h,d)&&(d.src?m._evalUrl&&m._evalUrl(d.src):m.globalEval((d.text||d.textContent||d.innerHTML||"").replace(qa,"")));i=c=null}return this}}),m.each({appendTo:"append",prependTo:"prepend",insertBefore:"before",insertAfter:"after",replaceAll:"replaceWith"},function(a,b){m.fn[a]=function(a){for(var c,d=0,e=[],g=m(a),h=g.length-1;h>=d;d++)c=d===h?this:this.clone(!0),m(g[d])[b](c),f.apply(e,c.get());return this.pushStack(e)}});var Ca,Da={};function Ea(b,c){var d,e=m(c.createElement(b)).appendTo(c.body),f=a.getDefaultComputedStyle&&(d=a.getDefaultComputedStyle(e[0]))?d.display:m.css(e[0],"display");return e.detach(),f}function Fa(a){var b=y,c=Da[a];return c||(c=Ea(a,b),"none"!==c&&c||(Ca=(Ca||m("<iframe frameborder='0' width='0' height='0'/>")).appendTo(b.documentElement),b=(Ca[0].contentWindow||Ca[0].contentDocument).document,b.write(),b.close(),c=Ea(a,b),Ca.detach()),Da[a]=c),c}!function(){var a;k.shrinkWrapBlocks=function(){if(null!=a)return a;a=!1;var b,c,d;return c=y.getElementsByTagName("body")[0],c&&c.style?(b=y.createElement("div"),d=y.createElement("div"),d.style.cssText="position:absolute;border:0;width:0;height:0;top:0;left:-9999px",c.appendChild(d).appendChild(b),typeof b.style.zoom!==K&&(b.style.cssText="-webkit-box-sizing:content-box;-moz-box-sizing:content-box;box-sizing:content-box;display:block;margin:0;border:0;padding:1px;width:1px;zoom:1",b.appendChild(y.createElement("div")).style.width="5px",a=3!==b.offsetWidth),c.removeChild(d),a):void 0}}();var Ga=/^margin/,Ha=new RegExp("^("+S+")(?!px)[a-z%]+$","i"),Ia,Ja,Ka=/^(top|right|bottom|left)$/;a.getComputedStyle?(Ia=function(b){return b.ownerDocument.defaultView.opener?b.ownerDocument.defaultView.getComputedStyle(b,null):a.getComputedStyle(b,null)},Ja=function(a,b,c){var d,e,f,g,h=a.style;return c=c||Ia(a),g=c?c.getPropertyValue(b)||c[b]:void 0,c&&(""!==g||m.contains(a.ownerDocument,a)||(g=m.style(a,b)),Ha.test(g)&&Ga.test(b)&&(d=h.width,e=h.minWidth,f=h.maxWidth,h.minWidth=h.maxWidth=h.width=g,g=c.width,h.width=d,h.minWidth=e,h.maxWidth=f)),void 0===g?g:g+""}):y.documentElement.currentStyle&&(Ia=function(a){return a.currentStyle},Ja=function(a,b,c){var d,e,f,g,h=a.style;return c=c||Ia(a),g=c?c[b]:void 0,null==g&&h&&h[b]&&(g=h[b]),Ha.test(g)&&!Ka.test(b)&&(d=h.left,e=a.runtimeStyle,f=e&&e.left,f&&(e.left=a.currentStyle.left),h.left="fontSize"===b?"1em":g,g=h.pixelLeft+"px",h.left=d,f&&(e.left=f)),void 0===g?g:g+""||"auto"});function La(a,b){return{get:function(){var c=a();if(null!=c)return c?void delete this.get:(this.get=b).apply(this,arguments)}}}!function(){var b,c,d,e,f,g,h;if(b=y.createElement("div"),b.innerHTML="  <link/><table></table><a href='/a'>a</a><input type='checkbox'/>",d=b.getElementsByTagName("a")[0],c=d&&d.style){c.cssText="float:left;opacity:.5",k.opacity="0.5"===c.opacity,k.cssFloat=!!c.cssFloat,b.style.backgroundClip="content-box",b.cloneNode(!0).style.backgroundClip="",k.clearCloneStyle="content-box"===b.style.backgroundClip,k.boxSizing=""===c.boxSizing||""===c.MozBoxSizing||""===c.WebkitBoxSizing,m.extend(k,{reliableHiddenOffsets:function(){return null==g&&i(),g},boxSizingReliable:function(){return null==f&&i(),f},pixelPosition:function(){return null==e&&i(),e},reliableMarginRight:function(){return null==h&&i(),h}});function i(){var b,c,d,i;c=y.getElementsByTagName("body")[0],c&&c.style&&(b=y.createElement("div"),d=y.createElement("div"),d.style.cssText="position:absolute;border:0;width:0;height:0;top:0;left:-9999px",c.appendChild(d).appendChild(b),b.style.cssText="-webkit-box-sizing:border-box;-moz-box-sizing:border-box;box-sizing:border-box;display:block;margin-top:1%;top:1%;border:1px;padding:1px;width:4px;position:absolute",e=f=!1,h=!0,a.getComputedStyle&&(e="1%"!==(a.getComputedStyle(b,null)||{}).top,f="4px"===(a.getComputedStyle(b,null)||{width:"4px"}).width,i=b.appendChild(y.createElement("div")),i.style.cssText=b.style.cssText="-webkit-box-sizing:content-box;-moz-box-sizing:content-box;box-sizing:content-box;display:block;margin:0;border:0;padding:0",i.style.marginRight=i.style.width="0",b.style.width="1px",h=!parseFloat((a.getComputedStyle(i,null)||{}).marginRight),b.removeChild(i)),b.innerHTML="<table><tr><td></td><td>t</td></tr></table>",i=b.getElementsByTagName("td"),i[0].style.cssText="margin:0;border:0;padding:0;display:none",g=0===i[0].offsetHeight,g&&(i[0].style.display="",i[1].style.display="none",g=0===i[0].offsetHeight),c.removeChild(d))}}}(),m.swap=function(a,b,c,d){var e,f,g={};for(f in b)g[f]=a.style[f],a.style[f]=b[f];e=c.apply(a,d||[]);for(f in b)a.style[f]=g[f];return e};var Ma=/alpha\([^)]*\)/i,Na=/opacity\s*=\s*([^)]*)/,Oa=/^(none|table(?!-c[ea]).+)/,Pa=new RegExp("^("+S+")(.*)$","i"),Qa=new RegExp("^([+-])=("+S+")","i"),Ra={position:"absolute",visibility:"hidden",display:"block"},Sa={letterSpacing:"0",fontWeight:"400"},Ta=["Webkit","O","Moz","ms"];function Ua(a,b){if(b in a)return b;var c=b.charAt(0).toUpperCase()+b.slice(1),d=b,e=Ta.length;while(e--)if(b=Ta[e]+c,b in a)return b;return d}function Va(a,b){for(var c,d,e,f=[],g=0,h=a.length;h>g;g++)d=a[g],d.style&&(f[g]=m._data(d,"olddisplay"),c=d.style.display,b?(f[g]||"none"!==c||(d.style.display=""),""===d.style.display&&U(d)&&(f[g]=m._data(d,"olddisplay",Fa(d.nodeName)))):(e=U(d),(c&&"none"!==c||!e)&&m._data(d,"olddisplay",e?c:m.css(d,"display"))));for(g=0;h>g;g++)d=a[g],d.style&&(b&&"none"!==d.style.display&&""!==d.style.display||(d.style.display=b?f[g]||"":"none"));return a}function Wa(a,b,c){var d=Pa.exec(b);return d?Math.max(0,d[1]-(c||0))+(d[2]||"px"):b}function Xa(a,b,c,d,e){for(var f=c===(d?"border":"content")?4:"width"===b?1:0,g=0;4>f;f+=2)"margin"===c&&(g+=m.css(a,c+T[f],!0,e)),d?("content"===c&&(g-=m.css(a,"padding"+T[f],!0,e)),"margin"!==c&&(g-=m.css(a,"border"+T[f]+"Width",!0,e))):(g+=m.css(a,"padding"+T[f],!0,e),"padding"!==c&&(g+=m.css(a,"border"+T[f]+"Width",!0,e)));return g}function Ya(a,b,c){var d=!0,e="width"===b?a.offsetWidth:a.offsetHeight,f=Ia(a),g=k.boxSizing&&"border-box"===m.css(a,"boxSizing",!1,f);if(0>=e||null==e){if(e=Ja(a,b,f),(0>e||null==e)&&(e=a.style[b]),Ha.test(e))return e;d=g&&(k.boxSizingReliable()||e===a.style[b]),e=parseFloat(e)||0}return e+Xa(a,b,c||(g?"border":"content"),d,f)+"px"}m.extend({cssHooks:{opacity:{get:function(a,b){if(b){var c=Ja(a,"opacity");return""===c?"1":c}}}},cssNumber:{columnCount:!0,fillOpacity:!0,flexGrow:!0,flexShrink:!0,fontWeight:!0,lineHeight:!0,opacity:!0,order:!0,orphans:!0,widows:!0,zIndex:!0,zoom:!0},cssProps:{"float":k.cssFloat?"cssFloat":"styleFloat"},style:function(a,b,c,d){if(a&&3!==a.nodeType&&8!==a.nodeType&&a.style){var e,f,g,h=m.camelCase(b),i=a.style;if(b=m.cssProps[h]||(m.cssProps[h]=Ua(i,h)),g=m.cssHooks[b]||m.cssHooks[h],void 0===c)return g&&"get"in g&&void 0!==(e=g.get(a,!1,d))?e:i[b];if(f=typeof c,"string"===f&&(e=Qa.exec(c))&&(c=(e[1]+1)*e[2]+parseFloat(m.css(a,b)),f="number"),null!=c&&c===c&&("number"!==f||m.cssNumber[h]||(c+="px"),k.clearCloneStyle||""!==c||0!==b.indexOf("background")||(i[b]="inherit"),!(g&&"set"in g&&void 0===(c=g.set(a,c,d)))))try{i[b]=c}catch(j){}}},css:function(a,b,c,d){var e,f,g,h=m.camelCase(b);return b=m.cssProps[h]||(m.cssProps[h]=Ua(a.style,h)),g=m.cssHooks[b]||m.cssHooks[h],g&&"get"in g&&(f=g.get(a,!0,c)),void 0===f&&(f=Ja(a,b,d)),"normal"===f&&b in Sa&&(f=Sa[b]),""===c||c?(e=parseFloat(f),c===!0||m.isNumeric(e)?e||0:f):f}}),m.each(["height","width"],function(a,b){m.cssHooks[b]={get:function(a,c,d){return c?Oa.test(m.css(a,"display"))&&0===a.offsetWidth?m.swap(a,Ra,function(){return Ya(a,b,d)}):Ya(a,b,d):void 0},set:function(a,c,d){var e=d&&Ia(a);return Wa(a,c,d?Xa(a,b,d,k.boxSizing&&"border-box"===m.css(a,"boxSizing",!1,e),e):0)}}}),k.opacity||(m.cssHooks.opacity={get:function(a,b){return Na.test((b&&a.currentStyle?a.currentStyle.filter:a.style.filter)||"")?.01*parseFloat(RegExp.$1)+"":b?"1":""},set:function(a,b){var c=a.style,d=a.currentStyle,e=m.isNumeric(b)?"alpha(opacity="+100*b+")":"",f=d&&d.filter||c.filter||"";c.zoom=1,(b>=1||""===b)&&""===m.trim(f.replace(Ma,""))&&c.removeAttribute&&(c.removeAttribute("filter"),""===b||d&&!d.filter)||(c.filter=Ma.test(f)?f.replace(Ma,e):f+" "+e)}}),m.cssHooks.marginRight=La(k.reliableMarginRight,function(a,b){return b?m.swap(a,{display:"inline-block"},Ja,[a,"marginRight"]):void 0}),m.each({margin:"",padding:"",border:"Width"},function(a,b){m.cssHooks[a+b]={expand:function(c){for(var d=0,e={},f="string"==typeof c?c.split(" "):[c];4>d;d++)e[a+T[d]+b]=f[d]||f[d-2]||f[0];return e}},Ga.test(a)||(m.cssHooks[a+b].set=Wa)}),m.fn.extend({css:function(a,b){return V(this,function(a,b,c){var d,e,f={},g=0;if(m.isArray(b)){for(d=Ia(a),e=b.length;e>g;g++)f[b[g]]=m.css(a,b[g],!1,d);return f}return void 0!==c?m.style(a,b,c):m.css(a,b)},a,b,arguments.length>1)},show:function(){return Va(this,!0)},hide:function(){return Va(this)},toggle:function(a){return"boolean"==typeof a?a?this.show():this.hide():this.each(function(){U(this)?m(this).show():m(this).hide()})}});function Za(a,b,c,d,e){
return new Za.prototype.init(a,b,c,d,e)}m.Tween=Za,Za.prototype={constructor:Za,init:function(a,b,c,d,e,f){this.elem=a,this.prop=c,this.easing=e||"swing",this.options=b,this.start=this.now=this.cur(),this.end=d,this.unit=f||(m.cssNumber[c]?"":"px")},cur:function(){var a=Za.propHooks[this.prop];return a&&a.get?a.get(this):Za.propHooks._default.get(this)},run:function(a){var b,c=Za.propHooks[this.prop];return this.options.duration?this.pos=b=m.easing[this.easing](a,this.options.duration*a,0,1,this.options.duration):this.pos=b=a,this.now=(this.end-this.start)*b+this.start,this.options.step&&this.options.step.call(this.elem,this.now,this),c&&c.set?c.set(this):Za.propHooks._default.set(this),this}},Za.prototype.init.prototype=Za.prototype,Za.propHooks={_default:{get:function(a){var b;return null==a.elem[a.prop]||a.elem.style&&null!=a.elem.style[a.prop]?(b=m.css(a.elem,a.prop,""),b&&"auto"!==b?b:0):a.elem[a.prop]},set:function(a){m.fx.step[a.prop]?m.fx.step[a.prop](a):a.elem.style&&(null!=a.elem.style[m.cssProps[a.prop]]||m.cssHooks[a.prop])?m.style(a.elem,a.prop,a.now+a.unit):a.elem[a.prop]=a.now}}},Za.propHooks.scrollTop=Za.propHooks.scrollLeft={set:function(a){a.elem.nodeType&&a.elem.parentNode&&(a.elem[a.prop]=a.now)}},m.easing={linear:function(a){return a},swing:function(a){return.5-Math.cos(a*Math.PI)/2}},m.fx=Za.prototype.init,m.fx.step={};var $a,_a,ab=/^(?:toggle|show|hide)$/,bb=new RegExp("^(?:([+-])=|)("+S+")([a-z%]*)$","i"),cb=/queueHooks$/,db=[ib],eb={"*":[function(a,b){var c=this.createTween(a,b),d=c.cur(),e=bb.exec(b),f=e&&e[3]||(m.cssNumber[a]?"":"px"),g=(m.cssNumber[a]||"px"!==f&&+d)&&bb.exec(m.css(c.elem,a)),h=1,i=20;if(g&&g[3]!==f){f=f||g[3],e=e||[],g=+d||1;do h=h||".5",g/=h,m.style(c.elem,a,g+f);while(h!==(h=c.cur()/d)&&1!==h&&--i)}return e&&(g=c.start=+g||+d||0,c.unit=f,c.end=e[1]?g+(e[1]+1)*e[2]:+e[2]),c}]};function fb(){return setTimeout(function(){$a=void 0}),$a=m.now()}function gb(a,b){var c,d={height:a},e=0;for(b=b?1:0;4>e;e+=2-b)c=T[e],d["margin"+c]=d["padding"+c]=a;return b&&(d.opacity=d.width=a),d}function hb(a,b,c){for(var d,e=(eb[b]||[]).concat(eb["*"]),f=0,g=e.length;g>f;f++)if(d=e[f].call(c,b,a))return d}function ib(a,b,c){var d,e,f,g,h,i,j,l,n=this,o={},p=a.style,q=a.nodeType&&U(a),r=m._data(a,"fxshow");c.queue||(h=m._queueHooks(a,"fx"),null==h.unqueued&&(h.unqueued=0,i=h.empty.fire,h.empty.fire=function(){h.unqueued||i()}),h.unqueued++,n.always(function(){n.always(function(){h.unqueued--,m.queue(a,"fx").length||h.empty.fire()})})),1===a.nodeType&&("height"in b||"width"in b)&&(c.overflow=[p.overflow,p.overflowX,p.overflowY],j=m.css(a,"display"),l="none"===j?m._data(a,"olddisplay")||Fa(a.nodeName):j,"inline"===l&&"none"===m.css(a,"float")&&(k.inlineBlockNeedsLayout&&"inline"!==Fa(a.nodeName)?p.zoom=1:p.display="inline-block")),c.overflow&&(p.overflow="hidden",k.shrinkWrapBlocks()||n.always(function(){p.overflow=c.overflow[0],p.overflowX=c.overflow[1],p.overflowY=c.overflow[2]}));for(d in b)if(e=b[d],ab.exec(e)){if(delete b[d],f=f||"toggle"===e,e===(q?"hide":"show")){if("show"!==e||!r||void 0===r[d])continue;q=!0}o[d]=r&&r[d]||m.style(a,d)}else j=void 0;if(m.isEmptyObject(o))"inline"===("none"===j?Fa(a.nodeName):j)&&(p.display=j);else{r?"hidden"in r&&(q=r.hidden):r=m._data(a,"fxshow",{}),f&&(r.hidden=!q),q?m(a).show():n.done(function(){m(a).hide()}),n.done(function(){var b;m._removeData(a,"fxshow");for(b in o)m.style(a,b,o[b])});for(d in o)g=hb(q?r[d]:0,d,n),d in r||(r[d]=g.start,q&&(g.end=g.start,g.start="width"===d||"height"===d?1:0))}}function jb(a,b){var c,d,e,f,g;for(c in a)if(d=m.camelCase(c),e=b[d],f=a[c],m.isArray(f)&&(e=f[1],f=a[c]=f[0]),c!==d&&(a[d]=f,delete a[c]),g=m.cssHooks[d],g&&"expand"in g){f=g.expand(f),delete a[d];for(c in f)c in a||(a[c]=f[c],b[c]=e)}else b[d]=e}function kb(a,b,c){var d,e,f=0,g=db.length,h=m.Deferred().always(function(){delete i.elem}),i=function(){if(e)return!1;for(var b=$a||fb(),c=Math.max(0,j.startTime+j.duration-b),d=c/j.duration||0,f=1-d,g=0,i=j.tweens.length;i>g;g++)j.tweens[g].run(f);return h.notifyWith(a,[j,f,c]),1>f&&i?c:(h.resolveWith(a,[j]),!1)},j=h.promise({elem:a,props:m.extend({},b),opts:m.extend(!0,{specialEasing:{}},c),originalProperties:b,originalOptions:c,startTime:$a||fb(),duration:c.duration,tweens:[],createTween:function(b,c){var d=m.Tween(a,j.opts,b,c,j.opts.specialEasing[b]||j.opts.easing);return j.tweens.push(d),d},stop:function(b){var c=0,d=b?j.tweens.length:0;if(e)return this;for(e=!0;d>c;c++)j.tweens[c].run(1);return b?h.resolveWith(a,[j,b]):h.rejectWith(a,[j,b]),this}}),k=j.props;for(jb(k,j.opts.specialEasing);g>f;f++)if(d=db[f].call(j,a,k,j.opts))return d;return m.map(k,hb,j),m.isFunction(j.opts.start)&&j.opts.start.call(a,j),m.fx.timer(m.extend(i,{elem:a,anim:j,queue:j.opts.queue})),j.progress(j.opts.progress).done(j.opts.done,j.opts.complete).fail(j.opts.fail).always(j.opts.always)}m.Animation=m.extend(kb,{tweener:function(a,b){m.isFunction(a)?(b=a,a=["*"]):a=a.split(" ");for(var c,d=0,e=a.length;e>d;d++)c=a[d],eb[c]=eb[c]||[],eb[c].unshift(b)},prefilter:function(a,b){b?db.unshift(a):db.push(a)}}),m.speed=function(a,b,c){var d=a&&"object"==typeof a?m.extend({},a):{complete:c||!c&&b||m.isFunction(a)&&a,duration:a,easing:c&&b||b&&!m.isFunction(b)&&b};return d.duration=m.fx.off?0:"number"==typeof d.duration?d.duration:d.duration in m.fx.speeds?m.fx.speeds[d.duration]:m.fx.speeds._default,(null==d.queue||d.queue===!0)&&(d.queue="fx"),d.old=d.complete,d.complete=function(){m.isFunction(d.old)&&d.old.call(this),d.queue&&m.dequeue(this,d.queue)},d},m.fn.extend({fadeTo:function(a,b,c,d){return this.filter(U).css("opacity",0).show().end().animate({opacity:b},a,c,d)},animate:function(a,b,c,d){var e=m.isEmptyObject(a),f=m.speed(b,c,d),g=function(){var b=kb(this,m.extend({},a),f);(e||m._data(this,"finish"))&&b.stop(!0)};return g.finish=g,e||f.queue===!1?this.each(g):this.queue(f.queue,g)},stop:function(a,b,c){var d=function(a){var b=a.stop;delete a.stop,b(c)};return"string"!=typeof a&&(c=b,b=a,a=void 0),b&&a!==!1&&this.queue(a||"fx",[]),this.each(function(){var b=!0,e=null!=a&&a+"queueHooks",f=m.timers,g=m._data(this);if(e)g[e]&&g[e].stop&&d(g[e]);else for(e in g)g[e]&&g[e].stop&&cb.test(e)&&d(g[e]);for(e=f.length;e--;)f[e].elem!==this||null!=a&&f[e].queue!==a||(f[e].anim.stop(c),b=!1,f.splice(e,1));(b||!c)&&m.dequeue(this,a)})},finish:function(a){return a!==!1&&(a=a||"fx"),this.each(function(){var b,c=m._data(this),d=c[a+"queue"],e=c[a+"queueHooks"],f=m.timers,g=d?d.length:0;for(c.finish=!0,m.queue(this,a,[]),e&&e.stop&&e.stop.call(this,!0),b=f.length;b--;)f[b].elem===this&&f[b].queue===a&&(f[b].anim.stop(!0),f.splice(b,1));for(b=0;g>b;b++)d[b]&&d[b].finish&&d[b].finish.call(this);delete c.finish})}}),m.each(["toggle","show","hide"],function(a,b){var c=m.fn[b];m.fn[b]=function(a,d,e){return null==a||"boolean"==typeof a?c.apply(this,arguments):this.animate(gb(b,!0),a,d,e)}}),m.each({slideDown:gb("show"),slideUp:gb("hide"),slideToggle:gb("toggle"),fadeIn:{opacity:"show"},fadeOut:{opacity:"hide"},fadeToggle:{opacity:"toggle"}},function(a,b){m.fn[a]=function(a,c,d){return this.animate(b,a,c,d)}}),m.timers=[],m.fx.tick=function(){var a,b=m.timers,c=0;for($a=m.now();c<b.length;c++)a=b[c],a()||b[c]!==a||b.splice(c--,1);b.length||m.fx.stop(),$a=void 0},m.fx.timer=function(a){m.timers.push(a),a()?m.fx.start():m.timers.pop()},m.fx.interval=13,m.fx.start=function(){_a||(_a=setInterval(m.fx.tick,m.fx.interval))},m.fx.stop=function(){clearInterval(_a),_a=null},m.fx.speeds={slow:600,fast:200,_default:400},m.fn.delay=function(a,b){return a=m.fx?m.fx.speeds[a]||a:a,b=b||"fx",this.queue(b,function(b,c){var d=setTimeout(b,a);c.stop=function(){clearTimeout(d)}})},function(){var a,b,c,d,e;b=y.createElement("div"),b.setAttribute("className","t"),b.innerHTML="  <link/><table></table><a href='/a'>a</a><input type='checkbox'/>",d=b.getElementsByTagName("a")[0],c=y.createElement("select"),e=c.appendChild(y.createElement("option")),a=b.getElementsByTagName("input")[0],d.style.cssText="top:1px",k.getSetAttribute="t"!==b.className,k.style=/top/.test(d.getAttribute("style")),k.hrefNormalized="/a"===d.getAttribute("href"),k.checkOn=!!a.value,k.optSelected=e.selected,k.enctype=!!y.createElement("form").enctype,c.disabled=!0,k.optDisabled=!e.disabled,a=y.createElement("input"),a.setAttribute("value",""),k.input=""===a.getAttribute("value"),a.value="t",a.setAttribute("type","radio"),k.radioValue="t"===a.value}();var lb=/\r/g;m.fn.extend({val:function(a){var b,c,d,e=this[0];{if(arguments.length)return d=m.isFunction(a),this.each(function(c){var e;1===this.nodeType&&(e=d?a.call(this,c,m(this).val()):a,null==e?e="":"number"==typeof e?e+="":m.isArray(e)&&(e=m.map(e,function(a){return null==a?"":a+""})),b=m.valHooks[this.type]||m.valHooks[this.nodeName.toLowerCase()],b&&"set"in b&&void 0!==b.set(this,e,"value")||(this.value=e))});if(e)return b=m.valHooks[e.type]||m.valHooks[e.nodeName.toLowerCase()],b&&"get"in b&&void 0!==(c=b.get(e,"value"))?c:(c=e.value,"string"==typeof c?c.replace(lb,""):null==c?"":c)}}}),m.extend({valHooks:{option:{get:function(a){var b=m.find.attr(a,"value");return null!=b?b:m.trim(m.text(a))}},select:{get:function(a){for(var b,c,d=a.options,e=a.selectedIndex,f="select-one"===a.type||0>e,g=f?null:[],h=f?e+1:d.length,i=0>e?h:f?e:0;h>i;i++)if(c=d[i],!(!c.selected&&i!==e||(k.optDisabled?c.disabled:null!==c.getAttribute("disabled"))||c.parentNode.disabled&&m.nodeName(c.parentNode,"optgroup"))){if(b=m(c).val(),f)return b;g.push(b)}return g},set:function(a,b){var c,d,e=a.options,f=m.makeArray(b),g=e.length;while(g--)if(d=e[g],m.inArray(m.valHooks.option.get(d),f)>=0)try{d.selected=c=!0}catch(h){d.scrollHeight}else d.selected=!1;return c||(a.selectedIndex=-1),e}}}}),m.each(["radio","checkbox"],function(){m.valHooks[this]={set:function(a,b){return m.isArray(b)?a.checked=m.inArray(m(a).val(),b)>=0:void 0}},k.checkOn||(m.valHooks[this].get=function(a){return null===a.getAttribute("value")?"on":a.value})});var mb,nb,ob=m.expr.attrHandle,pb=/^(?:checked|selected)$/i,qb=k.getSetAttribute,rb=k.input;m.fn.extend({attr:function(a,b){return V(this,m.attr,a,b,arguments.length>1)},removeAttr:function(a){return this.each(function(){m.removeAttr(this,a)})}}),m.extend({attr:function(a,b,c){var d,e,f=a.nodeType;if(a&&3!==f&&8!==f&&2!==f)return typeof a.getAttribute===K?m.prop(a,b,c):(1===f&&m.isXMLDoc(a)||(b=b.toLowerCase(),d=m.attrHooks[b]||(m.expr.match.bool.test(b)?nb:mb)),void 0===c?d&&"get"in d&&null!==(e=d.get(a,b))?e:(e=m.find.attr(a,b),null==e?void 0:e):null!==c?d&&"set"in d&&void 0!==(e=d.set(a,c,b))?e:(a.setAttribute(b,c+""),c):void m.removeAttr(a,b))},removeAttr:function(a,b){var c,d,e=0,f=b&&b.match(E);if(f&&1===a.nodeType)while(c=f[e++])d=m.propFix[c]||c,m.expr.match.bool.test(c)?rb&&qb||!pb.test(c)?a[d]=!1:a[m.camelCase("default-"+c)]=a[d]=!1:m.attr(a,c,""),a.removeAttribute(qb?c:d)},attrHooks:{type:{set:function(a,b){if(!k.radioValue&&"radio"===b&&m.nodeName(a,"input")){var c=a.value;return a.setAttribute("type",b),c&&(a.value=c),b}}}}}),nb={set:function(a,b,c){return b===!1?m.removeAttr(a,c):rb&&qb||!pb.test(c)?a.setAttribute(!qb&&m.propFix[c]||c,c):a[m.camelCase("default-"+c)]=a[c]=!0,c}},m.each(m.expr.match.bool.source.match(/\w+/g),function(a,b){var c=ob[b]||m.find.attr;ob[b]=rb&&qb||!pb.test(b)?function(a,b,d){var e,f;return d||(f=ob[b],ob[b]=e,e=null!=c(a,b,d)?b.toLowerCase():null,ob[b]=f),e}:function(a,b,c){return c?void 0:a[m.camelCase("default-"+b)]?b.toLowerCase():null}}),rb&&qb||(m.attrHooks.value={set:function(a,b,c){return m.nodeName(a,"input")?void(a.defaultValue=b):mb&&mb.set(a,b,c)}}),qb||(mb={set:function(a,b,c){var d=a.getAttributeNode(c);return d||a.setAttributeNode(d=a.ownerDocument.createAttribute(c)),d.value=b+="","value"===c||b===a.getAttribute(c)?b:void 0}},ob.id=ob.name=ob.coords=function(a,b,c){var d;return c?void 0:(d=a.getAttributeNode(b))&&""!==d.value?d.value:null},m.valHooks.button={get:function(a,b){var c=a.getAttributeNode(b);return c&&c.specified?c.value:void 0},set:mb.set},m.attrHooks.contenteditable={set:function(a,b,c){mb.set(a,""===b?!1:b,c)}},m.each(["width","height"],function(a,b){m.attrHooks[b]={set:function(a,c){return""===c?(a.setAttribute(b,"auto"),c):void 0}}})),k.style||(m.attrHooks.style={get:function(a){return a.style.cssText||void 0},set:function(a,b){return a.style.cssText=b+""}});var sb=/^(?:input|select|textarea|button|object)$/i,tb=/^(?:a|area)$/i;m.fn.extend({prop:function(a,b){return V(this,m.prop,a,b,arguments.length>1)},removeProp:function(a){return a=m.propFix[a]||a,this.each(function(){try{this[a]=void 0,delete this[a]}catch(b){}})}}),m.extend({propFix:{"for":"htmlFor","class":"className"},prop:function(a,b,c){var d,e,f,g=a.nodeType;if(a&&3!==g&&8!==g&&2!==g)return f=1!==g||!m.isXMLDoc(a),f&&(b=m.propFix[b]||b,e=m.propHooks[b]),void 0!==c?e&&"set"in e&&void 0!==(d=e.set(a,c,b))?d:a[b]=c:e&&"get"in e&&null!==(d=e.get(a,b))?d:a[b]},propHooks:{tabIndex:{get:function(a){var b=m.find.attr(a,"tabindex");return b?parseInt(b,10):sb.test(a.nodeName)||tb.test(a.nodeName)&&a.href?0:-1}}}}),k.hrefNormalized||m.each(["href","src"],function(a,b){m.propHooks[b]={get:function(a){return a.getAttribute(b,4)}}}),k.optSelected||(m.propHooks.selected={get:function(a){var b=a.parentNode;return b&&(b.selectedIndex,b.parentNode&&b.parentNode.selectedIndex),null}}),m.each(["tabIndex","readOnly","maxLength","cellSpacing","cellPadding","rowSpan","colSpan","useMap","frameBorder","contentEditable"],function(){m.propFix[this.toLowerCase()]=this}),k.enctype||(m.propFix.enctype="encoding");var ub=/[\t\r\n\f]/g;m.fn.extend({addClass:function(a){var b,c,d,e,f,g,h=0,i=this.length,j="string"==typeof a&&a;if(m.isFunction(a))return this.each(function(b){m(this).addClass(a.call(this,b,this.className))});if(j)for(b=(a||"").match(E)||[];i>h;h++)if(c=this[h],d=1===c.nodeType&&(c.className?(" "+c.className+" ").replace(ub," "):" ")){f=0;while(e=b[f++])d.indexOf(" "+e+" ")<0&&(d+=e+" ");g=m.trim(d),c.className!==g&&(c.className=g)}return this},removeClass:function(a){var b,c,d,e,f,g,h=0,i=this.length,j=0===arguments.length||"string"==typeof a&&a;if(m.isFunction(a))return this.each(function(b){m(this).removeClass(a.call(this,b,this.className))});if(j)for(b=(a||"").match(E)||[];i>h;h++)if(c=this[h],d=1===c.nodeType&&(c.className?(" "+c.className+" ").replace(ub," "):"")){f=0;while(e=b[f++])while(d.indexOf(" "+e+" ")>=0)d=d.replace(" "+e+" "," ");g=a?m.trim(d):"",c.className!==g&&(c.className=g)}return this},toggleClass:function(a,b){var c=typeof a;return"boolean"==typeof b&&"string"===c?b?this.addClass(a):this.removeClass(a):this.each(m.isFunction(a)?function(c){m(this).toggleClass(a.call(this,c,this.className,b),b)}:function(){if("string"===c){var b,d=0,e=m(this),f=a.match(E)||[];while(b=f[d++])e.hasClass(b)?e.removeClass(b):e.addClass(b)}else(c===K||"boolean"===c)&&(this.className&&m._data(this,"__className__",this.className),this.className=this.className||a===!1?"":m._data(this,"__className__")||"")})},hasClass:function(a){for(var b=" "+a+" ",c=0,d=this.length;d>c;c++)if(1===this[c].nodeType&&(" "+this[c].className+" ").replace(ub," ").indexOf(b)>=0)return!0;return!1}}),m.each("blur focus focusin focusout load resize scroll unload click dblclick mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave change select submit keydown keypress keyup error contextmenu".split(" "),function(a,b){m.fn[b]=function(a,c){return arguments.length>0?this.on(b,null,a,c):this.trigger(b)}}),m.fn.extend({hover:function(a,b){return this.mouseenter(a).mouseleave(b||a)},bind:function(a,b,c){return this.on(a,null,b,c)},unbind:function(a,b){return this.off(a,null,b)},delegate:function(a,b,c,d){return this.on(b,a,c,d)},undelegate:function(a,b,c){return 1===arguments.length?this.off(a,"**"):this.off(b,a||"**",c)}});var vb=m.now(),wb=/\?/,xb=/(,)|(\[|{)|(}|])|"(?:[^"\\\r\n]|\\["\\\/bfnrt]|\\u[\da-fA-F]{4})*"\s*:?|true|false|null|-?(?!0\d)\d+(?:\.\d+|)(?:[eE][+-]?\d+|)/g;m.parseJSON=function(b){if(a.JSON&&a.JSON.parse)return a.JSON.parse(b+"");var c,d=null,e=m.trim(b+"");return e&&!m.trim(e.replace(xb,function(a,b,e,f){return c&&b&&(d=0),0===d?a:(c=e||b,d+=!f-!e,"")}))?Function("return "+e)():m.error("Invalid JSON: "+b)},m.parseXML=function(b){var c,d;if(!b||"string"!=typeof b)return null;try{a.DOMParser?(d=new DOMParser,c=d.parseFromString(b,"text/xml")):(c=new ActiveXObject("Microsoft.XMLDOM"),c.async="false",c.loadXML(b))}catch(e){c=void 0}return c&&c.documentElement&&!c.getElementsByTagName("parsererror").length||m.error("Invalid XML: "+b),c};var yb,zb,Ab=/#.*$/,Bb=/([?&])_=[^&]*/,Cb=/^(.*?):[ \t]*([^\r\n]*)\r?$/gm,Db=/^(?:about|app|app-storage|.+-extension|file|res|widget):$/,Eb=/^(?:GET|HEAD)$/,Fb=/^\/\//,Gb=/^([\w.+-]+:)(?:\/\/(?:[^\/?#]*@|)([^\/?#:]*)(?::(\d+)|)|)/,Hb={},Ib={},Jb="*/".concat("*");try{zb=location.href}catch(Kb){zb=y.createElement("a"),zb.href="",zb=zb.href}yb=Gb.exec(zb.toLowerCase())||[];function Lb(a){return function(b,c){"string"!=typeof b&&(c=b,b="*");var d,e=0,f=b.toLowerCase().match(E)||[];if(m.isFunction(c))while(d=f[e++])"+"===d.charAt(0)?(d=d.slice(1)||"*",(a[d]=a[d]||[]).unshift(c)):(a[d]=a[d]||[]).push(c)}}function Mb(a,b,c,d){var e={},f=a===Ib;function g(h){var i;return e[h]=!0,m.each(a[h]||[],function(a,h){var j=h(b,c,d);return"string"!=typeof j||f||e[j]?f?!(i=j):void 0:(b.dataTypes.unshift(j),g(j),!1)}),i}return g(b.dataTypes[0])||!e["*"]&&g("*")}function Nb(a,b){var c,d,e=m.ajaxSettings.flatOptions||{};for(d in b)void 0!==b[d]&&((e[d]?a:c||(c={}))[d]=b[d]);return c&&m.extend(!0,a,c),a}function Ob(a,b,c){var d,e,f,g,h=a.contents,i=a.dataTypes;while("*"===i[0])i.shift(),void 0===e&&(e=a.mimeType||b.getResponseHeader("Content-Type"));if(e)for(g in h)if(h[g]&&h[g].test(e)){i.unshift(g);break}if(i[0]in c)f=i[0];else{for(g in c){if(!i[0]||a.converters[g+" "+i[0]]){f=g;break}d||(d=g)}f=f||d}return f?(f!==i[0]&&i.unshift(f),c[f]):void 0}function Pb(a,b,c,d){var e,f,g,h,i,j={},k=a.dataTypes.slice();if(k[1])for(g in a.converters)j[g.toLowerCase()]=a.converters[g];f=k.shift();while(f)if(a.responseFields[f]&&(c[a.responseFields[f]]=b),!i&&d&&a.dataFilter&&(b=a.dataFilter(b,a.dataType)),i=f,f=k.shift())if("*"===f)f=i;else if("*"!==i&&i!==f){if(g=j[i+" "+f]||j["* "+f],!g)for(e in j)if(h=e.split(" "),h[1]===f&&(g=j[i+" "+h[0]]||j["* "+h[0]])){g===!0?g=j[e]:j[e]!==!0&&(f=h[0],k.unshift(h[1]));break}if(g!==!0)if(g&&a["throws"])b=g(b);else try{b=g(b)}catch(l){return{state:"parsererror",error:g?l:"No conversion from "+i+" to "+f}}}return{state:"success",data:b}}m.extend({active:0,lastModified:{},etag:{},ajaxSettings:{url:zb,type:"GET",isLocal:Db.test(yb[1]),global:!0,processData:!0,async:!0,contentType:"application/x-www-form-urlencoded; charset=UTF-8",accepts:{"*":Jb,text:"text/plain",html:"text/html",xml:"application/xml, text/xml",json:"application/json, text/javascript"},contents:{xml:/xml/,html:/html/,json:/json/},responseFields:{xml:"responseXML",text:"responseText",json:"responseJSON"},converters:{"* text":String,"text html":!0,"text json":m.parseJSON,"text xml":m.parseXML},flatOptions:{url:!0,context:!0}},ajaxSetup:function(a,b){return b?Nb(Nb(a,m.ajaxSettings),b):Nb(m.ajaxSettings,a)},ajaxPrefilter:Lb(Hb),ajaxTransport:Lb(Ib),ajax:function(a,b){"object"==typeof a&&(b=a,a=void 0),b=b||{};var c,d,e,f,g,h,i,j,k=m.ajaxSetup({},b),l=k.context||k,n=k.context&&(l.nodeType||l.jquery)?m(l):m.event,o=m.Deferred(),p=m.Callbacks("once memory"),q=k.statusCode||{},r={},s={},t=0,u="canceled",v={readyState:0,getResponseHeader:function(a){var b;if(2===t){if(!j){j={};while(b=Cb.exec(f))j[b[1].toLowerCase()]=b[2]}b=j[a.toLowerCase()]}return null==b?null:b},getAllResponseHeaders:function(){return 2===t?f:null},setRequestHeader:function(a,b){var c=a.toLowerCase();return t||(a=s[c]=s[c]||a,r[a]=b),this},overrideMimeType:function(a){return t||(k.mimeType=a),this},statusCode:function(a){var b;if(a)if(2>t)for(b in a)q[b]=[q[b],a[b]];else v.always(a[v.status]);return this},abort:function(a){var b=a||u;return i&&i.abort(b),x(0,b),this}};if(o.promise(v).complete=p.add,v.success=v.done,v.error=v.fail,k.url=((a||k.url||zb)+"").replace(Ab,"").replace(Fb,yb[1]+"//"),k.type=b.method||b.type||k.method||k.type,k.dataTypes=m.trim(k.dataType||"*").toLowerCase().match(E)||[""],null==k.crossDomain&&(c=Gb.exec(k.url.toLowerCase()),k.crossDomain=!(!c||c[1]===yb[1]&&c[2]===yb[2]&&(c[3]||("http:"===c[1]?"80":"443"))===(yb[3]||("http:"===yb[1]?"80":"443")))),k.data&&k.processData&&"string"!=typeof k.data&&(k.data=m.param(k.data,k.traditional)),Mb(Hb,k,b,v),2===t)return v;h=m.event&&k.global,h&&0===m.active++&&m.event.trigger("ajaxStart"),k.type=k.type.toUpperCase(),k.hasContent=!Eb.test(k.type),e=k.url,k.hasContent||(k.data&&(e=k.url+=(wb.test(e)?"&":"?")+k.data,delete k.data),k.cache===!1&&(k.url=Bb.test(e)?e.replace(Bb,"$1_="+vb++):e+(wb.test(e)?"&":"?")+"_="+vb++)),k.ifModified&&(m.lastModified[e]&&v.setRequestHeader("If-Modified-Since",m.lastModified[e]),m.etag[e]&&v.setRequestHeader("If-None-Match",m.etag[e])),(k.data&&k.hasContent&&k.contentType!==!1||b.contentType)&&v.setRequestHeader("Content-Type",k.contentType),v.setRequestHeader("Accept",k.dataTypes[0]&&k.accepts[k.dataTypes[0]]?k.accepts[k.dataTypes[0]]+("*"!==k.dataTypes[0]?", "+Jb+"; q=0.01":""):k.accepts["*"]);for(d in k.headers)v.setRequestHeader(d,k.headers[d]);if(k.beforeSend&&(k.beforeSend.call(l,v,k)===!1||2===t))return v.abort();u="abort";for(d in{success:1,error:1,complete:1})v[d](k[d]);if(i=Mb(Ib,k,b,v)){v.readyState=1,h&&n.trigger("ajaxSend",[v,k]),k.async&&k.timeout>0&&(g=setTimeout(function(){v.abort("timeout")},k.timeout));try{t=1,i.send(r,x)}catch(w){if(!(2>t))throw w;x(-1,w)}}else x(-1,"No Transport");function x(a,b,c,d){var j,r,s,u,w,x=b;2!==t&&(t=2,g&&clearTimeout(g),i=void 0,f=d||"",v.readyState=a>0?4:0,j=a>=200&&300>a||304===a,c&&(u=Ob(k,v,c)),u=Pb(k,u,v,j),j?(k.ifModified&&(w=v.getResponseHeader("Last-Modified"),w&&(m.lastModified[e]=w),w=v.getResponseHeader("etag"),w&&(m.etag[e]=w)),204===a||"HEAD"===k.type?x="nocontent":304===a?x="notmodified":(x=u.state,r=u.data,s=u.error,j=!s)):(s=x,(a||!x)&&(x="error",0>a&&(a=0))),v.status=a,v.statusText=(b||x)+"",j?o.resolveWith(l,[r,x,v]):o.rejectWith(l,[v,x,s]),v.statusCode(q),q=void 0,h&&n.trigger(j?"ajaxSuccess":"ajaxError",[v,k,j?r:s]),p.fireWith(l,[v,x]),h&&(n.trigger("ajaxComplete",[v,k]),--m.active||m.event.trigger("ajaxStop")))}return v},getJSON:function(a,b,c){return m.get(a,b,c,"json")},getScript:function(a,b){return m.get(a,void 0,b,"script")}}),m.each(["get","post"],function(a,b){m[b]=function(a,c,d,e){return m.isFunction(c)&&(e=e||d,d=c,c=void 0),m.ajax({url:a,type:b,dataType:e,data:c,success:d})}}),m._evalUrl=function(a){return m.ajax({url:a,type:"GET",dataType:"script",async:!1,global:!1,"throws":!0})},m.fn.extend({wrapAll:function(a){if(m.isFunction(a))return this.each(function(b){m(this).wrapAll(a.call(this,b))});if(this[0]){var b=m(a,this[0].ownerDocument).eq(0).clone(!0);this[0].parentNode&&b.insertBefore(this[0]),b.map(function(){var a=this;while(a.firstChild&&1===a.firstChild.nodeType)a=a.firstChild;return a}).append(this)}return this},wrapInner:function(a){return this.each(m.isFunction(a)?function(b){m(this).wrapInner(a.call(this,b))}:function(){var b=m(this),c=b.contents();c.length?c.wrapAll(a):b.append(a)})},wrap:function(a){var b=m.isFunction(a);return this.each(function(c){m(this).wrapAll(b?a.call(this,c):a)})},unwrap:function(){return this.parent().each(function(){m.nodeName(this,"body")||m(this).replaceWith(this.childNodes)}).end()}}),m.expr.filters.hidden=function(a){return a.offsetWidth<=0&&a.offsetHeight<=0||!k.reliableHiddenOffsets()&&"none"===(a.style&&a.style.display||m.css(a,"display"))},m.expr.filters.visible=function(a){return!m.expr.filters.hidden(a)};var Qb=/%20/g,Rb=/\[\]$/,Sb=/\r?\n/g,Tb=/^(?:submit|button|image|reset|file)$/i,Ub=/^(?:input|select|textarea|keygen)/i;function Vb(a,b,c,d){var e;if(m.isArray(b))m.each(b,function(b,e){c||Rb.test(a)?d(a,e):Vb(a+"["+("object"==typeof e?b:"")+"]",e,c,d)});else if(c||"object"!==m.type(b))d(a,b);else for(e in b)Vb(a+"["+e+"]",b[e],c,d)}m.param=function(a,b){var c,d=[],e=function(a,b){b=m.isFunction(b)?b():null==b?"":b,d[d.length]=encodeURIComponent(a)+"="+encodeURIComponent(b)};if(void 0===b&&(b=m.ajaxSettings&&m.ajaxSettings.traditional),m.isArray(a)||a.jquery&&!m.isPlainObject(a))m.each(a,function(){e(this.name,this.value)});else for(c in a)Vb(c,a[c],b,e);return d.join("&").replace(Qb,"+")},m.fn.extend({serialize:function(){return m.param(this.serializeArray())},serializeArray:function(){return this.map(function(){var a=m.prop(this,"elements");return a?m.makeArray(a):this}).filter(function(){var a=this.type;return this.name&&!m(this).is(":disabled")&&Ub.test(this.nodeName)&&!Tb.test(a)&&(this.checked||!W.test(a))}).map(function(a,b){var c=m(this).val();return null==c?null:m.isArray(c)?m.map(c,function(a){return{name:b.name,value:a.replace(Sb,"\r\n")}}):{name:b.name,value:c.replace(Sb,"\r\n")}}).get()}}),m.ajaxSettings.xhr=void 0!==a.ActiveXObject?function(){return!this.isLocal&&/^(get|post|head|put|delete|options)$/i.test(this.type)&&Zb()||$b()}:Zb;var Wb=0,Xb={},Yb=m.ajaxSettings.xhr();a.attachEvent&&a.attachEvent("onunload",function(){for(var a in Xb)Xb[a](void 0,!0)}),k.cors=!!Yb&&"withCredentials"in Yb,Yb=k.ajax=!!Yb,Yb&&m.ajaxTransport(function(a){if(!a.crossDomain||k.cors){var b;return{send:function(c,d){var e,f=a.xhr(),g=++Wb;if(f.open(a.type,a.url,a.async,a.username,a.password),a.xhrFields)for(e in a.xhrFields)f[e]=a.xhrFields[e];a.mimeType&&f.overrideMimeType&&f.overrideMimeType(a.mimeType),a.crossDomain||c["X-Requested-With"]||(c["X-Requested-With"]="XMLHttpRequest");for(e in c)void 0!==c[e]&&f.setRequestHeader(e,c[e]+"");f.send(a.hasContent&&a.data||null),b=function(c,e){var h,i,j;if(b&&(e||4===f.readyState))if(delete Xb[g],b=void 0,f.onreadystatechange=m.noop,e)4!==f.readyState&&f.abort();else{j={},h=f.status,"string"==typeof f.responseText&&(j.text=f.responseText);try{i=f.statusText}catch(k){i=""}h||!a.isLocal||a.crossDomain?1223===h&&(h=204):h=j.text?200:404}j&&d(h,i,j,f.getAllResponseHeaders())},a.async?4===f.readyState?setTimeout(b):f.onreadystatechange=Xb[g]=b:b()},abort:function(){b&&b(void 0,!0)}}}});function Zb(){try{return new a.XMLHttpRequest}catch(b){}}function $b(){try{return new a.ActiveXObject("Microsoft.XMLHTTP")}catch(b){}}m.ajaxSetup({accepts:{script:"text/javascript, application/javascript, application/ecmascript, application/x-ecmascript"},contents:{script:/(?:java|ecma)script/},converters:{"text script":function(a){return m.globalEval(a),a}}}),m.ajaxPrefilter("script",function(a){void 0===a.cache&&(a.cache=!1),a.crossDomain&&(a.type="GET",a.global=!1)}),m.ajaxTransport("script",function(a){if(a.crossDomain){var b,c=y.head||m("head")[0]||y.documentElement;return{send:function(d,e){b=y.createElement("script"),b.async=!0,a.scriptCharset&&(b.charset=a.scriptCharset),b.src=a.url,b.onload=b.onreadystatechange=function(a,c){(c||!b.readyState||/loaded|complete/.test(b.readyState))&&(b.onload=b.onreadystatechange=null,b.parentNode&&b.parentNode.removeChild(b),b=null,c||e(200,"success"))},c.insertBefore(b,c.firstChild)},abort:function(){b&&b.onload(void 0,!0)}}}});var _b=[],ac=/(=)\?(?=&|$)|\?\?/;m.ajaxSetup({jsonp:"callback",jsonpCallback:function(){var a=_b.pop()||m.expando+"_"+vb++;return this[a]=!0,a}}),m.ajaxPrefilter("json jsonp",function(b,c,d){var e,f,g,h=b.jsonp!==!1&&(ac.test(b.url)?"url":"string"==typeof b.data&&!(b.contentType||"").indexOf("application/x-www-form-urlencoded")&&ac.test(b.data)&&"data");return h||"jsonp"===b.dataTypes[0]?(e=b.jsonpCallback=m.isFunction(b.jsonpCallback)?b.jsonpCallback():b.jsonpCallback,h?b[h]=b[h].replace(ac,"$1"+e):b.jsonp!==!1&&(b.url+=(wb.test(b.url)?"&":"?")+b.jsonp+"="+e),b.converters["script json"]=function(){return g||m.error(e+" was not called"),g[0]},b.dataTypes[0]="json",f=a[e],a[e]=function(){g=arguments},d.always(function(){a[e]=f,b[e]&&(b.jsonpCallback=c.jsonpCallback,_b.push(e)),g&&m.isFunction(f)&&f(g[0]),g=f=void 0}),"script"):void 0}),m.parseHTML=function(a,b,c){if(!a||"string"!=typeof a)return null;"boolean"==typeof b&&(c=b,b=!1),b=b||y;var d=u.exec(a),e=!c&&[];return d?[b.createElement(d[1])]:(d=m.buildFragment([a],b,e),e&&e.length&&m(e).remove(),m.merge([],d.childNodes))};var bc=m.fn.load;m.fn.load=function(a,b,c){if("string"!=typeof a&&bc)return bc.apply(this,arguments);var d,e,f,g=this,h=a.indexOf(" ");return h>=0&&(d=m.trim(a.slice(h,a.length)),a=a.slice(0,h)),m.isFunction(b)?(c=b,b=void 0):b&&"object"==typeof b&&(f="POST"),g.length>0&&m.ajax({url:a,type:f,dataType:"html",data:b}).done(function(a){e=arguments,g.html(d?m("<div>").append(m.parseHTML(a)).find(d):a)}).complete(c&&function(a,b){g.each(c,e||[a.responseText,b,a])}),this},m.each(["ajaxStart","ajaxStop","ajaxComplete","ajaxError","ajaxSuccess","ajaxSend"],function(a,b){m.fn[b]=function(a){return this.on(b,a)}}),m.expr.filters.animated=function(a){return m.grep(m.timers,function(b){return a===b.elem}).length};var cc=a.document.documentElement;function dc(a){return m.isWindow(a)?a:9===a.nodeType?a.defaultView||a.parentWindow:!1}m.offset={setOffset:function(a,b,c){var d,e,f,g,h,i,j,k=m.css(a,"position"),l=m(a),n={};"static"===k&&(a.style.position="relative"),h=l.offset(),f=m.css(a,"top"),i=m.css(a,"left"),j=("absolute"===k||"fixed"===k)&&m.inArray("auto",[f,i])>-1,j?(d=l.position(),g=d.top,e=d.left):(g=parseFloat(f)||0,e=parseFloat(i)||0),m.isFunction(b)&&(b=b.call(a,c,h)),null!=b.top&&(n.top=b.top-h.top+g),null!=b.left&&(n.left=b.left-h.left+e),"using"in b?b.using.call(a,n):l.css(n)}},m.fn.extend({offset:function(a){if(arguments.length)return void 0===a?this:this.each(function(b){m.offset.setOffset(this,a,b)});var b,c,d={top:0,left:0},e=this[0],f=e&&e.ownerDocument;if(f)return b=f.documentElement,m.contains(b,e)?(typeof e.getBoundingClientRect!==K&&(d=e.getBoundingClientRect()),c=dc(f),{top:d.top+(c.pageYOffset||b.scrollTop)-(b.clientTop||0),left:d.left+(c.pageXOffset||b.scrollLeft)-(b.clientLeft||0)}):d},position:function(){if(this[0]){var a,b,c={top:0,left:0},d=this[0];return"fixed"===m.css(d,"position")?b=d.getBoundingClientRect():(a=this.offsetParent(),b=this.offset(),m.nodeName(a[0],"html")||(c=a.offset()),c.top+=m.css(a[0],"borderTopWidth",!0),c.left+=m.css(a[0],"borderLeftWidth",!0)),{top:b.top-c.top-m.css(d,"marginTop",!0),left:b.left-c.left-m.css(d,"marginLeft",!0)}}},offsetParent:function(){return this.map(function(){var a=this.offsetParent||cc;while(a&&!m.nodeName(a,"html")&&"static"===m.css(a,"position"))a=a.offsetParent;return a||cc})}}),m.each({scrollLeft:"pageXOffset",scrollTop:"pageYOffset"},function(a,b){var c=/Y/.test(b);m.fn[a]=function(d){return V(this,function(a,d,e){var f=dc(a);return void 0===e?f?b in f?f[b]:f.document.documentElement[d]:a[d]:void(f?f.scrollTo(c?m(f).scrollLeft():e,c?e:m(f).scrollTop()):a[d]=e)},a,d,arguments.length,null)}}),m.each(["top","left"],function(a,b){m.cssHooks[b]=La(k.pixelPosition,function(a,c){return c?(c=Ja(a,b),Ha.test(c)?m(a).position()[b]+"px":c):void 0})}),m.each({Height:"height",Width:"width"},function(a,b){m.each({padding:"inner"+a,content:b,"":"outer"+a},function(c,d){m.fn[d]=function(d,e){var f=arguments.length&&(c||"boolean"!=typeof d),g=c||(d===!0||e===!0?"margin":"border");return V(this,function(b,c,d){var e;return m.isWindow(b)?b.document.documentElement["client"+a]:9===b.nodeType?(e=b.documentElement,Math.max(b.body["scroll"+a],e["scroll"+a],b.body["offset"+a],e["offset"+a],e["client"+a])):void 0===d?m.css(b,c,g):m.style(b,c,d,g)},b,f?d:void 0,f,null)}})}),m.fn.size=function(){return this.length},m.fn.andSelf=m.fn.addBack,"function"==typeof define&&define.amd&&define("jquery",[],function(){return m});var ec=a.jQuery,fc=a.$;return m.noConflict=function(b){return a.$===m&&(a.$=fc),b&&a.jQuery===m&&(a.jQuery=ec),m},typeof b===K&&(a.jQuery=a.$=m),m});
</script>
<style type="text/css">
.container-fluid.crosstalk-bscols {
margin-left: -30px;
margin-right: -30px;
white-space: normal;
}

body > .container-fluid.crosstalk-bscols {
margin-left: auto;
margin-right: auto;
}
.crosstalk-input-checkboxgroup .crosstalk-options-group .crosstalk-options-column {
display: inline-block;
padding-right: 12px;
vertical-align: top;
}
@media only screen and (max-width:480px) {
.crosstalk-input-checkboxgroup .crosstalk-options-group .crosstalk-options-column {
display: block;
padding-right: inherit;
}
}
</style>
<script>!function a(b,c,d){function e(g,h){if(!c[g]){if(!b[g]){var i="function"==typeof require&&require;if(!h&&i)return i(g,!0);if(f)return f(g,!0);var j=new Error("Cannot find module '"+g+"'");throw j.code="MODULE_NOT_FOUND",j}var k=c[g]={exports:{}};b[g][0].call(k.exports,function(a){var c=b[g][1][a];return e(c?c:a)},k,k.exports,a,b,c,d)}return c[g].exports}for(var f="function"==typeof require&&require,g=0;g<d.length;g++)e(d[g]);return e}({1:[function(a,b,c){"use strict";function d(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}Object.defineProperty(c,"__esModule",{value:!0});var e=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),f=function(){function a(){d(this,a),this._types={},this._seq=0}return e(a,[{key:"on",value:function(a,b){var c=this._types[a];c||(c=this._types[a]={});var d="sub"+this._seq++;return c[d]=b,d}},{key:"off",value:function(a,b){var c=this._types[a];if("function"==typeof b){for(var d in c)if(c.hasOwnProperty(d)&&c[d]===b)return delete c[d],d;return!1}if("string"==typeof b)return!(!c||!c[b])&&(delete c[b],b);throw new Error("Unexpected type for listener")}},{key:"trigger",value:function(a,b,c){var d=this._types[a];for(var e in d)d.hasOwnProperty(e)&&d[e].call(c,b)}}]),a}();c.default=f},{}],2:[function(a,b,c){"use strict";function d(a){if(a&&a.__esModule)return a;var b={};if(null!=a)for(var c in a)Object.prototype.hasOwnProperty.call(a,c)&&(b[c]=a[c]);return b.default=a,b}function e(a){return a&&a.__esModule?a:{default:a}}function f(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}function g(a){var b=a.var("filterset"),c=b.get();return c||(c=new m.default,b.set(c)),c}function h(){return r++}Object.defineProperty(c,"__esModule",{value:!0}),c.FilterHandle=void 0;var i=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),j=a("./events"),k=e(j),l=a("./filterset"),m=e(l),n=a("./group"),o=e(n),p=a("./util"),q=d(p),r=1;c.FilterHandle=function(){function a(b,c){f(this,a),this._eventRelay=new k.default,this._emitter=new q.SubscriptionTracker(this._eventRelay),this._group=null,this._filterSet=null,this._filterVar=null,this._varOnChangeSub=null,this._extraInfo=q.extend({sender:this},c),this._id="filter"+h(),this.setGroup(b)}return i(a,[{key:"setGroup",value:function(a){var b=this;if(this._group!==a&&(this._group||a)&&(this._filterVar&&(this._filterVar.off("change",this._varOnChangeSub),this.clear(),this._varOnChangeSub=null,this._filterVar=null,this._filterSet=null),this._group=a,a)){a=(0,o.default)(a),this._filterSet=g(a),this._filterVar=(0,o.default)(a).var("filter");var c=this._filterVar.on("change",function(a){b._eventRelay.trigger("change",a,b)});this._varOnChangeSub=c}}},{key:"_mergeExtraInfo",value:function(a){return q.extend({},this._extraInfo?this._extraInfo:null,a?a:null)}},{key:"close",value:function(){this._emitter.removeAllListeners(),this.clear(),this.setGroup(null)}},{key:"clear",value:function(a){this._filterSet&&(this._filterSet.clear(this._id),this._onChange(a))}},{key:"set",value:function(a,b){this._filterSet&&(this._filterSet.update(this._id,a),this._onChange(b))}},{key:"on",value:function(a,b){return this._emitter.on(a,b)}},{key:"off",value:function(a,b){return this._emitter.off(a,b)}},{key:"_onChange",value:function(a){this._filterSet&&this._filterVar.set(this._filterSet.value,this._mergeExtraInfo(a))}},{key:"filteredKeys",get:function(){return this._filterSet?this._filterSet.value:null}}]),a}()},{"./events":1,"./filterset":3,"./group":4,"./util":11}],3:[function(a,b,c){"use strict";function d(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}function e(a,b){return a===b?0:a<b?-1:a>b?1:void 0}Object.defineProperty(c,"__esModule",{value:!0});var f=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),g=a("./util"),h=function(){function a(){d(this,a),this.reset()}return f(a,[{key:"reset",value:function(){this._handles={},this._keys={},this._value=null,this._activeHandles=0}},{key:"update",value:function(a,b){null!==b&&(b=b.slice(0),b.sort(e));var c=(0,g.diffSortedLists)(this._handles[a],b),d=c.added,f=c.removed;this._handles[a]=b;for(var h=0;h<d.length;h++)this._keys[d[h]]=(this._keys[d[h]]||0)+1;for(var i=0;i<f.length;i++)this._keys[f[i]]--;this._updateValue(b)}},{key:"_updateValue",value:function(){var a=arguments.length>0&&void 0!==arguments[0]?arguments[0]:this._allKeys,b=Object.keys(this._handles).length;if(0===b)this._value=null;else{this._value=[];for(var c=0;c<a.length;c++){var d=this._keys[a[c]];d===b&&this._value.push(a[c])}}}},{key:"clear",value:function(a){if("undefined"!=typeof this._handles[a]){var b=this._handles[a];b||(b=[]);for(var c=0;c<b.length;c++)this._keys[b[c]]--;delete this._handles[a],this._updateValue()}}},{key:"value",get:function(){return this._value}},{key:"_allKeys",get:function(){var a=Object.keys(this._keys);return a.sort(e),a}}]),a}();c.default=h},{"./util":11}],4:[function(a,b,c){(function(b){"use strict";function d(a){return a&&a.__esModule?a:{default:a}}function e(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}function f(a){if(a&&"string"==typeof a)return k.hasOwnProperty(a)||(k[a]=new l(a)),k[a];if("object"===("undefined"==typeof a?"undefined":h(a))&&a._vars&&a.var)return a;if(Array.isArray(a)&&1==a.length&&"string"==typeof a[0])return f(a[0]);throw new Error("Invalid groupName argument")}Object.defineProperty(c,"__esModule",{value:!0});var g=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),h="function"==typeof Symbol&&"symbol"==typeof Symbol.iterator?function(a){return typeof a}:function(a){return a&&"function"==typeof Symbol&&a.constructor===Symbol&&a!==Symbol.prototype?"symbol":typeof a};c.default=f;var i=a("./var"),j=d(i);b.__crosstalk_groups=b.__crosstalk_groups||{};var k=b.__crosstalk_groups,l=function(){function a(b){e(this,a),this.name=b,this._vars={}}return g(a,[{key:"var",value:function(a){if(!a||"string"!=typeof a)throw new Error("Invalid var name");return this._vars.hasOwnProperty(a)||(this._vars[a]=new j.default(this,a)),this._vars[a]}},{key:"has",value:function(a){if(!a||"string"!=typeof a)throw new Error("Invalid var name");return this._vars.hasOwnProperty(a)}}]),a}()}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./var":12}],5:[function(a,b,c){(function(b){"use strict";function d(a){return a&&a.__esModule?a:{default:a}}function e(a){return k.var(a)}function f(a){return k.has(a)}Object.defineProperty(c,"__esModule",{value:!0});var g=a("./group"),h=d(g),i=a("./selection"),j=a("./filter");a("./input"),a("./input_selectize"),a("./input_checkboxgroup"),a("./input_slider");var k=(0,h.default)("default");b.Shiny&&b.Shiny.addCustomMessageHandler("update-client-value",function(a){"string"==typeof a.group?(0,h.default)(a.group).var(a.name).set(a.value):e(a.name).set(a.value)});var l={group:h.default,var:e,has:f,SelectionHandle:i.SelectionHandle,FilterHandle:j.FilterHandle};c.default=l,b.crosstalk=l}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./filter":2,"./group":4,"./input":6,"./input_checkboxgroup":7,"./input_selectize":8,"./input_slider":9,"./selection":10}],6:[function(a,b,c){(function(a){"use strict";function b(b){i[b.className]=b,a.document&&"complete"!==a.document.readyState?h(function(){d()}):a.document&&setTimeout(d,100)}function d(){Object.keys(i).forEach(function(a){var b=i[a];h("."+b.className).not(".crosstalk-input-bound").each(function(a,c){g(b,c)})})}function e(a){return a.replace(/([!"#$%&'()*+,.\/:;<=>?@\[\\\]^`{|}~])/g,"\\$1")}function f(a){var b=h(a);Object.keys(i).forEach(function(c){if(b.hasClass(c)&&!b.hasClass("crosstalk-input-bound")){var d=i[c];g(d,a)}})}function g(a,b){var c=h(b).find("script[type='application/json'][data-for='"+e(b.id)+"']"),d=JSON.parse(c[0].innerText),f=a.factory(b,d);h(b).data("crosstalk-instance",f),h(b).addClass("crosstalk-input-bound")}Object.defineProperty(c,"__esModule",{value:!0}),c.register=b;var h=a.jQuery,i={};a.Shiny&&!function(){var b=new a.Shiny.InputBinding,c=a.jQuery;c.extend(b,{find:function(a){return c(a).find(".crosstalk-input")},initialize:function(a){c(a).hasClass("crosstalk-input-bound")||f(a)},getId:function(a){return a.id},getValue:function(a){},setValue:function(a,b){},receiveMessage:function(a,b){},subscribe:function(a,b){c(a).data("crosstalk-instance").resume()},unsubscribe:function(a){c(a).data("crosstalk-instance").suspend()}}),a.Shiny.inputBindings.register(b,"crosstalk.inputBinding")}()}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{}],7:[function(a,b,c){(function(b){"use strict";function c(a){if(a&&a.__esModule)return a;var b={};if(null!=a)for(var c in a)Object.prototype.hasOwnProperty.call(a,c)&&(b[c]=a[c]);return b.default=a,b}var d=a("./input"),e=c(d),f=a("./filter"),g=b.jQuery;e.register({className:"crosstalk-input-checkboxgroup",factory:function(a,b){var c=new f.FilterHandle(b.group),d=void 0,e=g(a);return e.on("change","input[type='checkbox']",function(){var a=e.find("input[type='checkbox']:checked");0===a.length?(d=null,c.clear()):!function(){var e={};a.each(function(){b.map[this.value].forEach(function(a){e[a]=!0})});var f=Object.keys(e);f.sort(),d=f,c.set(f)}()}),{suspend:function(){c.clear()},resume:function(){d&&c.set(d)}}}})}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./filter":2,"./input":6}],8:[function(a,b,c){(function(b){"use strict";function c(a){if(a&&a.__esModule)return a;var b={};if(null!=a)for(var c in a)Object.prototype.hasOwnProperty.call(a,c)&&(b[c]=a[c]);return b.default=a,b}var d=a("./input"),e=c(d),f=a("./util"),g=c(f),h=a("./filter"),i=b.jQuery;e.register({className:"crosstalk-input-select",factory:function(a,b){var c=[{value:"",label:"(All)"}],d=g.dataframeToD3(b.items),e={options:c.concat(d),valueField:"value",labelField:"label",searchField:"label"},f=i(a).find("select")[0],j=i(f).selectize(e)[0].selectize,k=new h.FilterHandle(b.group),l=void 0;return j.on("change",function(){0===j.items.length?(l=null,k.clear()):!function(){var a={};j.items.forEach(function(c){b.map[c].forEach(function(b){a[b]=!0})});var c=Object.keys(a);c.sort(),l=c,k.set(c)}()}),{suspend:function(){k.clear()},resume:function(){l&&k.set(l)}}}})}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./filter":2,"./input":6,"./util":11}],9:[function(a,b,c){(function(b){"use strict";function c(a){if(a&&a.__esModule)return a;var b={};if(null!=a)for(var c in a)Object.prototype.hasOwnProperty.call(a,c)&&(b[c]=a[c]);return b.default=a,b}function d(a,b){for(var c=a.toString();c.length<b;)c="0"+c;return c}function e(a){return a instanceof Date?a.getUTCFullYear()+"-"+d(a.getUTCMonth()+1,2)+"-"+d(a.getUTCDate(),2):null}var f=function(){function a(a,b){var c=[],d=!0,e=!1,f=void 0;try{for(var g,h=a[Symbol.iterator]();!(d=(g=h.next()).done)&&(c.push(g.value),!b||c.length!==b);d=!0);}catch(a){e=!0,f=a}finally{try{!d&&h.return&&h.return()}finally{if(e)throw f}}return c}return function(b,c){if(Array.isArray(b))return b;if(Symbol.iterator in Object(b))return a(b,c);throw new TypeError("Invalid attempt to destructure non-iterable instance")}}(),g=a("./input"),h=c(g),i=a("./filter"),j=b.jQuery,k=b.strftime;h.register({className:"crosstalk-input-slider",factory:function(a,b){function c(){var a=h.data("ionRangeSlider").result,b=void 0,c=h.data("data-type");return b="date"===c?function(a){return e(new Date(+a))}:"datetime"===c?function(a){return+a/1e3}:function(a){return+a},"double"===h.data("ionRangeSlider").options.type?[b(a.from),b(a.to)]:b(a.from)}var d=new i.FilterHandle(b.group),g={},h=j(a).find("input"),l=h.data("data-type"),m=h.data("time-format"),n=void 0;if("date"===l)n=k.utc(),g.prettify=function(a){return n(m,new Date(a))};else if("datetime"===l){var o=h.data("timezone");n=o?k.timezone(o):k,g.prettify=function(a){return n(m,new Date(a))}}h.ionRangeSlider(g);var p=null;return h.on("change.crosstalkSliderInput",function(a){if(!h.data("updating")&&!h.data("animating")){for(var e=c(),g=f(e,2),i=g[0],j=g[1],k=[],l=0;l<b.values.length;l++){var m=b.values[l];m>=i&&m<=j&&k.push(b.keys[l])}k.sort(),d.set(k),p=k}}),{suspend:function(){d.clear()},resume:function(){p&&d.set(p)}}}})}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./filter":2,"./input":6}],10:[function(a,b,c){"use strict";function d(a){if(a&&a.__esModule)return a;var b={};if(null!=a)for(var c in a)Object.prototype.hasOwnProperty.call(a,c)&&(b[c]=a[c]);return b.default=a,b}function e(a){return a&&a.__esModule?a:{default:a}}function f(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}Object.defineProperty(c,"__esModule",{value:!0}),c.SelectionHandle=void 0;var g=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),h=a("./events"),i=e(h),j=a("./group"),k=e(j),l=a("./util"),m=d(l);c.SelectionHandle=function(){function a(){var b=arguments.length>0&&void 0!==arguments[0]?arguments[0]:null,c=arguments.length>1&&void 0!==arguments[1]?arguments[1]:null;f(this,a),this._eventRelay=new i.default,this._emitter=new m.SubscriptionTracker(this._eventRelay),this._group=null,this._var=null,this._varOnChangeSub=null,this._extraInfo=m.extend({sender:this},c),this.setGroup(b)}return g(a,[{key:"setGroup",value:function(a){var b=this;if(this._group!==a&&(this._group||a)&&(this._var&&(this._var.off("change",this._varOnChangeSub),this._var=null,this._varOnChangeSub=null),this._group=a,a)){this._var=(0,k.default)(a).var("selection");var c=this._var.on("change",function(a){b._eventRelay.trigger("change",a,b)});this._varOnChangeSub=c}}},{key:"_mergeExtraInfo",value:function(a){return m.extend({},this._extraInfo?this._extraInfo:null,a?a:null)}},{key:"set",value:function(a,b){this._var&&this._var.set(a,this._mergeExtraInfo(b))}},{key:"clear",value:function(a){this._var&&this.set(void 0,this._mergeExtraInfo(a))}},{key:"on",value:function(a,b){return this._emitter.on(a,b)}},{key:"off",value:function(a,b){return this._emitter.off(a,b)}},{key:"close",value:function(){this._emitter.removeAllListeners(),this.setGroup(null)}},{key:"value",get:function(){return this._var?this._var.get():null}}]),a}()},{"./events":1,"./group":4,"./util":11}],11:[function(a,b,c){"use strict";function d(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}function e(a){for(var b=arguments.length,c=Array(b>1?b-1:0),d=1;d<b;d++)c[d-1]=arguments[d];for(var e=0;e<c.length;e++){var f=c[e];if("undefined"!=typeof f&&null!==f)for(var g in f)f.hasOwnProperty(g)&&(a[g]=f[g])}return a}function f(a){for(var b=1;b<a.length;b++)if(a[b]<=a[b-1])throw new Error("List is not sorted or contains duplicate")}function g(a,b){var c=0,d=0;a||(a=[]),b||(b=[]);var e=[],g=[];for(f(a),f(b);c<a.length&&d<b.length;)a[c]===b[d]?(c++,d++):a[c]<b[d]?e.push(a[c++]):g.push(b[d++]);return c<a.length&&(e=e.concat(a.slice(c))),d<b.length&&(g=g.concat(b.slice(d))),{removed:e,added:g}}function h(a){var b=[],c=void 0;for(var d in a){if(a.hasOwnProperty(d)&&b.push(d),"object"!==j(a[d])||"undefined"==typeof a[d].length)throw new Error("All fields must be arrays");if("undefined"!=typeof c&&c!==a[d].length)throw new Error("All fields must be arrays of the same length");c=a[d].length}for(var e=[],f=void 0,g=0;g<c;g++){f={};for(var h=0;h<b.length;h++)f[b[h]]=a[b[h]][g];e.push(f)}return e}Object.defineProperty(c,"__esModule",{value:!0});var i=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),j="function"==typeof Symbol&&"symbol"==typeof Symbol.iterator?function(a){return typeof a}:function(a){return a&&"function"==typeof Symbol&&a.constructor===Symbol&&a!==Symbol.prototype?"symbol":typeof a};c.extend=e,c.checkSorted=f,c.diffSortedLists=g,c.dataframeToD3=h;c.SubscriptionTracker=function(){function a(b){d(this,a),this._emitter=b,this._subs={}}return i(a,[{key:"on",value:function(a,b){var c=this._emitter.on(a,b);return this._subs[c]=a,c}},{key:"off",value:function(a,b){var c=this._emitter.off(a,b);return c&&delete this._subs[c],c}},{key:"removeAllListeners",value:function(){var a=this,b=this._subs;this._subs={},Object.keys(b).forEach(function(c){a._emitter.off(b[c],c)})}}]),a}()},{}],12:[function(a,b,c){(function(b){"use strict";function d(a){return a&&a.__esModule?a:{default:a}}function e(a,b){if(!(a instanceof b))throw new TypeError("Cannot call a class as a function")}Object.defineProperty(c,"__esModule",{value:!0});var f="function"==typeof Symbol&&"symbol"==typeof Symbol.iterator?function(a){return typeof a}:function(a){return a&&"function"==typeof Symbol&&a.constructor===Symbol&&a!==Symbol.prototype?"symbol":typeof a},g=function(){function a(a,b){for(var c=0;c<b.length;c++){var d=b[c];d.enumerable=d.enumerable||!1,d.configurable=!0,"value"in d&&(d.writable=!0),Object.defineProperty(a,d.key,d)}}return function(b,c,d){return c&&a(b.prototype,c),d&&a(b,d),b}}(),h=a("./events"),i=d(h),j=function(){function a(b,c,d){e(this,a),this._group=b,this._name=c,this._value=d,this._events=new i.default}return g(a,[{key:"get",value:function(){return this._value}},{key:"set",value:function(a,c){if(this._value!==a){var d=this._value;this._value=a;var e={};if(c&&"object"===("undefined"==typeof c?"undefined":f(c)))for(var g in c)c.hasOwnProperty(g)&&(e[g]=c[g]);e.oldValue=d,e.value=a,this._events.trigger("change",e,this),b.Shiny&&b.Shiny.onInputChange&&b.Shiny.onInputChange(".clientValue-"+(null!==this._group.name?this._group.name+"-":"")+this._name,"undefined"==typeof a?null:a)}}},{key:"on",value:function(a,b){return this._events.on(a,b)}},{key:"off",value:function(a,b){return this._events.off(a,b)}}]),a}();c.default=j}).call(this,"undefined"!=typeof global?global:"undefined"!=typeof self?self:"undefined"!=typeof window?window:{})},{"./events":1}]},{},[5]);
//# sourceMappingURL=crosstalk.min.js.map</script>
<style type="text/css">
slide:not(.current) .plotly.html-widget{
display: none;
}
</style>
<script>/**
* plotly.js v1.49.4
* Copyright 2012-2019, Plotly, Inc.
* All rights reserved.
* Licensed under the MIT license
*/
</head>
<body style="background-color: white;">
<div id="htmlwidget_container">
<div id="htmlwidget-8db415bae83108bb813f" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-8db415bae83108bb813f">{"x":{"data":[{"x":[267315,267316,267317,267318,267319,267320,267321,267322,267323,267324,267325,267326,267327,284042,284043,284044,284045,284046,284047,284048,284049,284050,284051,284052,284053,284054,284055,284056,284057,284058,284059,284060,284061,284062,284063,267796,267797,267798,267800,267801,267803,267804,267805,267807,267809,267810,267811,267812,267813,267814,267815,267817,266185,266187,266189,266190,266192,266193,266196,266197,266198,266199,266200,266201,266202,266203,266204,266205,266206,266207,266208,266209,266210,266211,271412,271413,271414,271415,271416,271418,271419,271420,271421,271422,271423,271425,271426,271427,271428,271429,271430,271431,271432,271433,271434,271435,271437,271438,271439,271440,271441,285697,285698,285699,285701,285702,285703,285704,285705,285707,285708,285709,285710,285713,285714,285715,285716,285717,285718,285719,285720,285721,285722,285724,285726,285727,285728,285729,277681,277682,277684,277686,277688,277690,277691,277692,277693,277694,277695,277697,277698,277699,277700,277702,277703,277704,277706,277707,277708,277709,277710,277712,277714,277716,269427,269428,269429,269430,269431,269432,269433,269437,269438,269439,269440,269441,269442,267101,267102,267103,267104,267105,267107,267108,267109,267110,267111,267112,267113,267114,267697,267698,267699,267700,267701,267702,280050,280051,280053,280054,280055,280056,280058,280059,280060,280062,280063,280064,280066,280067,280068,264086,264088,264089,264090,264091,264092,264093,264094,264095,264096,264097,264099,264100,264102,264103,264104,264105,264106,284542,284543,284544,284545,284546,284547,284548,284549,284550,284551,284553,284554,284556,268997,269001,269002,269004,269005,269006,269007,269008,269009,269010,269011,269013,269015,269016,269017,269018,269019,269020,269022,269023,269226,269227,269229,269230,269231,269234,269235,269237,269238,269240,269241,269242,269243,269244,269245,269246,269247,269248,269250,269251,269252,269253,269254,269256,269257,271739,271740,271742,271743,271744,271745,271747,271748,271749,271750,271752,271753,271754,271755,271756,271758,271759,271761,271762,271764,271766,271767,271769,271770,268714,268715,268716,268717,268718,268719,268720,268721,268722,268724,268725,268726,268727,268728,268729,268730,268731,268732,268733,268734,268736,268737,268738,268739,268740,268741,268742,268744,268745,265695,265696,265698,265699,265700,265701,265702,265703,265704,265706,265707,265708,265710,265712,277570,277571,277573,277574,277575,277576,277577,277578,277580,277581,277582,277584,277585,277586,277587,277588,277589,277590,277591,277593,277594,277599,286487,286488,286489,286490,286491,286492,286493,286496,286497,286499,286500,286501,286502,286504,286505,286506,278745,278746,278747,278748,278749,278750,278751,278752,278753,278754,278755,278756,278757,278759,278760,278761,278764,278765,278766,278767,278768,280123,280125,280126,280127,280128,280130,280131,280132,280133,280134,280135,280136,280137,280139,280140,280141,280142,280143,280144,264174,264175,264176,264177,264178,264179,264181,264182,264184,264185,264186,264187,264188,264189,264190,264191,264192,264194,264195,264197,264198,275283,275284,275285,275288,275289,275290,275291,275292,275293,275294,275295,275296,275298,275299,275300,275301,275302,275303,275304,275305,275306,275307,275308,275309,275310,275311,282949,282952,282954,282955,282957,282958,282959,282960,282961,282962,282963,282965,282967,282968,282970,282972,282973,282974,282975,282976,282977,282978,282979,282981,282982,282984,266619,266620,266621,266622,266623,266624,266626,266627,266628,266629,266632,266633,266634,266635,266637,266638,266639,266640,266641,266642,266643,266644,266645,266646,266647,266648,266649,266650,266651,266652,266653,266654,266657,269348,269349,269350,269351,269352,269353,269354,269355,269356,269357,269359,269360,269361,269362,269363,269364,269365,269366,285096,285097,285098,285099,285100,285101,285102,285104,285105,285106,285107,285108,285109,285110,285111,285112,285113,285114,276241,276242,276243,276245,276247,276248,276249,276250,276251,276252,276253,276254,276255,276256,276257,276258,276259,284812,284813,284814,284815,284816,284817,284818,284820,284821,284822,284823,284824,284825,284826,284827,284829,269539,269540,269541,269542,269544,269545,269546,269547,269548,269549,269550,269551,269552,269553,269554,269555,269556,269558,269559,269560,269561,269562,269563,276149,276150,276151,276153,276155,276157,276158,276159,276160,276161,276162,276165,276166,276167,276168,276169,276170,276171,276172,280908,280909,280912,280913,280914,280916,280917,280918,280919,280920,280921,280922,280923,280924,280925,280927,280928,280930,280931,280932,280933,280934,280935,280936,280939,280941,280942,280943,280944,282478,282480,282481,282482,282483,282484,282485,282487,282488,282489,282490,282491,282494,282495,282497,285386,285387,285388,285390,285391,285392,285393,285394,285395,285396,285397,285398,285399,285400,285401,285402,285403,285404,285405,285406,285407,285408,285409,285410,285411],"y":[-350.682171,378.881703,-359.585714,-282.728045,-499.079365,-345.45648,-380.438642,-120.186352,-351.365239,305.182857,301.027972,355.324324,262.238095,-244.93419,-96.492248,391.257576,-295.506849,380.326371,393.668621,524.620482,421.658263,397.995283,-317.442997,454.734286,-321.164619,-239.092983,-255.995327,-152.206704,-296.063218,-520.901857,-496.593583,-364.085918,-251.624625,483.260416,342.552346,-553.88674,-343.903846,-766.923488,-158.390728,-168.79288,-168.399224,-228.856716,-340.402573,-390.159794,-276.915464,-476.981405,-329.420421,-367.307229,-175.193548,-274.610604,-372.912121,-443.392524,-229,-227.780347,-316.903966,-318.010345,-390.64151,-106.13928,-124.490196,-178.062351,-271.787425,-281.250597,-252.779891,-227.204762,-200.544199,-200.843299,-192.586,-182.59542,-302.539306,-690.226006,-537.844961,-291.395034,-155.037901,-538.912181,-682.664078,-894.854219,-157.176079,-341.698925,-214.833333,-108.885922,-261.662609,-198.801471,-444.268939,-303.86445,-164.514424,-317.574132,-271.413367,279.445652,-180,-178.731052,-327.820827,-207.208053,-154.202817,-205.503837,-271.703463,-168.655257,-333.95691,-405.31124,-358.116099,-202.9225,-370.100204,350.906746,-441.733606,-292.991453,-213.214567,-251.223684,-670.217229,-265.83965,-175.905263,-197,-295.556548,-239.14882,-195.7,-1011.674858,-235.306773,-243.245354,173.631579,-261.744856,-100.875,-165.046404,-302.048872,-141.279177,-192.966723,-132.645161,-118.106383,-337.929398,-357.220588,-189.560302,335.943262,367.764865,-148.829268,-217.862944,-308.794922,-374.008196,-377.197674,-168.24812,-229.81087,-437.565022,-410.222222,-386.642105,-594.213368,-384.939189,-265.483606,294.705882,-283.761905,-182.494792,-171.423868,-273.121694,-134.20436,-236.68543,-1017.786008,-234.005495,-134.538251,-398.166667,-413.5,-267.378641,201.569002,-170.856305,302.328912,536.807771,-253.955326,-378.373866,-304.876033,-236.721232,-363.540609,410.245455,658.273504,-134.520482,-407.495345,-323.910364,-363.578151,-855.19603,-301.039823,-169.700837,-228.639258,474.487952,-309,354.896774,847.72449,-329,448.460481,404.398844,428.719243,497.994671,344.191781,476.472131,451.61139,537.181818,567.823219,554.679214,452.462604,-250.884861,120.604743,-365.51884,411.994231,-282.546296,-228.125964,-533.390099,-432.570833,442.571429,-166.403637,-705.388998,-298.856624,-375.191276,-88.006431,-358.86828,-355.262537,-93.759312,-693.79845,-345.289575,-475.60817,-600.997811,-143.903646,-526.82766,-158.177778,-419.350561,-271.137931,-513.160417,-259.528112,-176.382096,-241.530337,-446.700258,-363.91498,-385.032934,-542.287726,-120.81932,-122.991896,-124.993939,-582.321508,-125.86503,-446.588028,-481.836364,-426.078534,-363.24846,-309.789981,-301.291089,-460.498925,-286.010593,-414.204633,-418.249371,-161.864993,-296.592274,-889.085152,-251.093023,-537.236302,-204.660377,-120.476666,-632.600806,-389.313883,-311.754258,-228.97543,555.046555,-194.543807,-239.958904,512.773421,-444.222467,-262.684326,-346.261682,-282.010292,-410.092784,-329.715447,-296.010937,-403.166241,-339.179872,-343.29927,-373.009934,-404.576687,-499.784172,-171.413223,-327.681382,-312.914706,-345.911168,-361.478365,-331.819355,-654.25098,-335.525478,-469,-317.684418,-297.081286,-175.332613,-386.743859,-576.345946,-324.491578,-438.100205,-264.605442,-748.456475,-140.957371,-692.550633,-240.798165,-383.793442,-396.207709,-263.826891,-529.834938,-95.667857,-309.738738,-239.57947,-327.592356,-560.944636,-340.213873,-306.516949,573.735849,-355.897322,-395.425605,-399.075145,-332.250784,-487.37968,-344.334988,-325.8379,-197.381068,-134.266666,-309.005235,-344.874016,-274.928218,-541.355372,-247.67347,-460.5,-259.375662,-243.440945,-145.16129,-534.602446,-207.381696,-479.95283,-173.089606,-93.587859,-317.343373,-293.608955,-417.027914,-293.377224,-204.105422,-194.567568,-242.459016,-385.656676,-780.050633,411.745098,-122.725,-346.648551,-429.710021,-667.067751,500.742857,-236.429072,-396.729729,-158.282805,-194.07571,-263.252595,-149.151234,-251.098859,-311.350848,-203.794872,-199.119186,-280.549669,-131.244444,-213.415144,-233.82,-339.406732,-201.605129,-154.134021,-232.3808,-540.843138,-361.668283,-230.49537,-169.051282,-386.432353,-116.938213,-405.268116,-258.807607,-174.117647,-306.449721,-253.469565,-248.059242,-409.511509,130.52,-257.131119,-392.172481,-241.442935,-271.02835,-158.190381,-407.479108,-518.132075,-360.21663,-355.014663,-451.133208,-443.300626,-286.758064,-214.25641,-181.917647,-228.395189,-595.218241,-344.284211,-343.756604,-551.680328,-248.540463,-228.802752,-207.658823,-380.639626,-138.5725,-297.243698,-95.806,-69.689256,-161.405325,-424.906833,-446.008621,-312.323741,-200.952381,-586.809931,-231.074433,-309.703947,-392.608939,-262.66432,-263.213675,-407.806358,-223.379859,-268.052083,-420.564516,-61.591022,-70.173374,-807.265775,-250.459746,-223.203662,-261.691684,-245.127778,-381.297297,-197.335715,263.783158,-794.377104,-308.004938,-421.354292,-310.632716,-224.537994,-314.116439,-231.44343,-429.43325,-199.663424,-376.232441,-236.583515,-202.142395,-148.673171,-324.878403,-109.514135,-406.157197,-192.192893,-334.702595,-326.278772,-115.047297,-122.071428,-193.704698,-378.342373,-173.710744,-178.03966,-312.276923,-315.170426,-418.069621,-312.92562,-291.37037,-131.84,-283.405316,-243.719299,-157.432585,-200.771052,-255.18091,-222.323899,-341.488294,-864.051359,254.407176,-270.009075,-272.918848,-660.509478,-102.501618,-233.975261,-453.892857,-251.114974,-195.765823,-328.013922,-527.91778,-228.978469,-380.379528,-237.844011,-142.385225,-473.491978,-154.056818,-190.573129,-120.544236,-357.884869,-275.159011,-126.199376,-125.362179,-228.200489,-558.77621,-158.761363,-436.681725,-226.62419,-309.622516,-321.640212,-398.4,-267.249337,-508.851563,-584.34584,-191.899554,-250.461818,-267.479109,259.808918,-489.400428,-274.98374,-220.007211,-285.345845,-176.820339,-289.659639,-177.306123,-240.276134,-408.461783,-289.907104,-71.7357509999999,-334.416667,-94.960423,-314.377162,-154.746312,-341.127059,-497.718421,-361.501597,-182.461538,-230.728683,-243.989517,-210.887538,-196.983553,-150.433963,-225.762019,-336.521472,-102.442822,-165.753521,-274.802667,-348.688932,-143.351293,-452.67347,-357.298343,-503.712708,-313.491762,-396.658385,-252.80367,-311.871508,-287.877984,-339.364687,-377.455414,-447.533333,-415.896907,-278.026357,-140.517482,-181.453377,-369.47929,-736.176101,-249.820423,-205.300147,-362.413476,-307.172489,-621.233216,-277.056926,-404.780538,-186.814737,-434.922395,-439.092308,-280.943894,-361.460123,-141.335897,-439.052055,-423.760656,-302.082677,-637.08867,-447.768708,-281.603279,-286.069767,-364.897527,-446.87703,-558.41577,-369.590504,-245.222841,-392.815508,-312.63286,-607.863481,-794.929448,-182.717461,-277.258403,-379.967105,-329.816626,-296.419118,-362.384844,-375.070881,-312.619048,-395.335416,-302.991044,-1033.495495,-100.858859,-248.667953,-235.247159,-299.253898,-351.441479,-226.408502,-198.932643,-519.826498,-235.880342,-120.105751,-288.017921,-347.66855,-429.519537,-272.174785,-349.223529,-156.906639,-338.7,-240.98,-839.424615,-455.73064,-525.446215,-591.580645,-139.842407,-194.316103,-375.746881,-303.904376,-218.286885,-196.185259,-402.963076,-216.517891,-308.410296,-228.147335,-249.850799,-272.604136,-513.797267,-376.197761,-528.27115,-452.161058,-277.098802,-502.113637,-242.181474,-323.375886,-158.752907,-195.502703,-97.851406,-117.204893,-416.748644,-106.671958,-294.813559,-451.517241,-253.402062,-561.699641,-555.309955,-104.662791,-472.787931,-253.221212,-756.0375,-257.115183,-471.608696,-428.456274,-327.341652,-264.706989,-183.454352,-292.501984,-284.635802,-282.608511,-468.518812,-301.7,-701.927308,-226.6,-355.328804,-316.285156,-326.553634,-92.301587,-116.26087,-173.666667,-232.714286,-458.965704,-195.955391,-356.89046,162.272727,-200.328846,-626.575646,-473.168761,-453.177694,-542.676768,-152.565341,-346.910448,-325.342916,-477.672673,-375.106464,-298.908356,-292.330049,-572.747747,-394.414331,-301.23192,-659.305054,-638.511785,-418.641196,-317.190931,-562.016842,-237.003725,-221.758364,-399.356401,-258.200758,-318.789969,-180.597101,-319.941278,-258.242745,-340.560096,-785.543611,-176.459948,-161.939339,-321.33721,-193.579881,-341.419183,-173.937662,-228.562,-288.208531,-449.383908],"text":["mapobject_id: 267315<br />detached_metric:  -350.68217<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267316<br />detached_metric:   378.88170<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267317<br />detached_metric:  -359.58571<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267318<br />detached_metric:  -282.72805<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267319<br />detached_metric:  -499.07936<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267320<br />detached_metric:  -345.45648<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267321<br />detached_metric:  -380.43864<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267322<br />detached_metric:  -120.18635<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267323<br />detached_metric:  -351.36524<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267324<br />detached_metric:   305.18286<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267325<br />detached_metric:   301.02797<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267326<br />detached_metric:   355.32432<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 267327<br />detached_metric:   262.23809<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 0","mapobject_id: 284042<br />detached_metric:  -244.93419<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284043<br />detached_metric:   -96.49225<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284044<br />detached_metric:   391.25758<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284045<br />detached_metric:  -295.50685<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284046<br />detached_metric:   380.32637<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284047<br />detached_metric:   393.66862<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284048<br />detached_metric:   524.62048<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284049<br />detached_metric:   421.65826<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284050<br />detached_metric:   397.99528<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284051<br />detached_metric:  -317.44300<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284052<br />detached_metric:   454.73429<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284053<br />detached_metric:  -321.16462<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284054<br />detached_metric:  -239.09298<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284055<br />detached_metric:  -255.99533<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284056<br />detached_metric:  -152.20670<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284057<br />detached_metric:  -296.06322<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284058<br />detached_metric:  -520.90186<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284059<br />detached_metric:  -496.59358<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284060<br />detached_metric:  -364.08592<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284061<br />detached_metric:  -251.62463<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284062<br />detached_metric:   483.26042<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 284063<br />detached_metric:   342.55235<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 0","mapobject_id: 267796<br />detached_metric:  -553.88674<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267797<br />detached_metric:  -343.90385<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267798<br />detached_metric:  -766.92349<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267800<br />detached_metric:  -158.39073<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267801<br />detached_metric:  -168.79288<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267803<br />detached_metric:  -168.39922<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267804<br />detached_metric:  -228.85672<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267805<br />detached_metric:  -340.40257<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267807<br />detached_metric:  -390.15979<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267809<br />detached_metric:  -276.91546<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267810<br />detached_metric:  -476.98141<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267811<br />detached_metric:  -329.42042<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267812<br />detached_metric:  -367.30723<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267813<br />detached_metric:  -175.19355<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267814<br />detached_metric:  -274.61060<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267815<br />detached_metric:  -372.91212<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 267817<br />detached_metric:  -443.39252<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 0","mapobject_id: 266185<br />detached_metric:  -229.00000<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266187<br />detached_metric:  -227.78035<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266189<br />detached_metric:  -316.90397<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266190<br />detached_metric:  -318.01035<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266192<br />detached_metric:  -390.64151<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266193<br />detached_metric:  -106.13928<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266196<br />detached_metric:  -124.49020<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266197<br />detached_metric:  -178.06235<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266198<br />detached_metric:  -271.78742<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266199<br />detached_metric:  -281.25060<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266200<br />detached_metric:  -252.77989<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266201<br />detached_metric:  -227.20476<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266202<br />detached_metric:  -200.54420<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266203<br />detached_metric:  -200.84330<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266204<br />detached_metric:  -192.58600<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266205<br />detached_metric:  -182.59542<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266206<br />detached_metric:  -302.53931<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266207<br />detached_metric:  -690.22601<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266208<br />detached_metric:  -537.84496<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266209<br />detached_metric:  -291.39503<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266210<br />detached_metric:  -155.03790<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 266211<br />detached_metric:  -538.91218<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 0","mapobject_id: 271412<br />detached_metric:  -682.66408<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271413<br />detached_metric:  -894.85422<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271414<br />detached_metric:  -157.17608<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271415<br />detached_metric:  -341.69892<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271416<br />detached_metric:  -214.83333<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271418<br />detached_metric:  -108.88592<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271419<br />detached_metric:  -261.66261<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271420<br />detached_metric:  -198.80147<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271421<br />detached_metric:  -444.26894<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271422<br />detached_metric:  -303.86445<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271423<br />detached_metric:  -164.51442<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271425<br />detached_metric:  -317.57413<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271426<br />detached_metric:  -271.41337<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271427<br />detached_metric:   279.44565<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271428<br />detached_metric:  -180.00000<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271429<br />detached_metric:  -178.73105<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271430<br />detached_metric:  -327.82083<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271431<br />detached_metric:  -207.20805<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271432<br />detached_metric:  -154.20282<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271433<br />detached_metric:  -205.50384<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271434<br />detached_metric:  -271.70346<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271435<br />detached_metric:  -168.65526<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271437<br />detached_metric:  -333.95691<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271438<br />detached_metric:  -405.31124<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271439<br />detached_metric:  -358.11610<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271440<br />detached_metric:  -202.92250<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 271441<br />detached_metric:  -370.10020<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 0","mapobject_id: 285697<br />detached_metric:   350.90675<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285698<br />detached_metric:  -441.73361<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285699<br />detached_metric:  -292.99145<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285701<br />detached_metric:  -213.21457<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285702<br />detached_metric:  -251.22368<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285703<br />detached_metric:  -670.21723<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285704<br />detached_metric:  -265.83965<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285705<br />detached_metric:  -175.90526<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285707<br />detached_metric:  -197.00000<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285708<br />detached_metric:  -295.55655<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285709<br />detached_metric:  -239.14882<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285710<br />detached_metric:  -195.70000<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285713<br />detached_metric: -1011.67486<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285714<br />detached_metric:  -235.30677<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285715<br />detached_metric:  -243.24535<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285716<br />detached_metric:   173.63158<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285717<br />detached_metric:  -261.74486<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285718<br />detached_metric:  -100.87500<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285719<br />detached_metric:  -165.04640<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285720<br />detached_metric:  -302.04887<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285721<br />detached_metric:  -141.27918<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285722<br />detached_metric:  -192.96672<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285724<br />detached_metric:  -132.64516<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285726<br />detached_metric:  -118.10638<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285727<br />detached_metric:  -337.92940<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285728<br />detached_metric:  -357.22059<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 285729<br />detached_metric:  -189.56030<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 1","mapobject_id: 277681<br />detached_metric:   335.94326<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277682<br />detached_metric:   367.76486<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277684<br />detached_metric:  -148.82927<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277686<br />detached_metric:  -217.86294<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277688<br />detached_metric:  -308.79492<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277690<br />detached_metric:  -374.00820<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277691<br />detached_metric:  -377.19767<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277692<br />detached_metric:  -168.24812<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277693<br />detached_metric:  -229.81087<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277694<br />detached_metric:  -437.56502<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277695<br />detached_metric:  -410.22222<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277697<br />detached_metric:  -386.64211<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277698<br />detached_metric:  -594.21337<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277699<br />detached_metric:  -384.93919<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277700<br />detached_metric:  -265.48361<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277702<br />detached_metric:   294.70588<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277703<br />detached_metric:  -283.76190<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277704<br />detached_metric:  -182.49479<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277706<br />detached_metric:  -171.42387<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277707<br />detached_metric:  -273.12169<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277708<br />detached_metric:  -134.20436<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277709<br />detached_metric:  -236.68543<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277710<br />detached_metric: -1017.78601<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277712<br />detached_metric:  -234.00549<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277714<br />detached_metric:  -134.53825<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 277716<br />detached_metric:  -398.16667<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 1","mapobject_id: 269427<br />detached_metric:  -413.50000<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269428<br />detached_metric:  -267.37864<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269429<br />detached_metric:   201.56900<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269430<br />detached_metric:  -170.85631<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269431<br />detached_metric:   302.32891<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269432<br />detached_metric:   536.80777<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269433<br />detached_metric:  -253.95533<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269437<br />detached_metric:  -378.37387<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269438<br />detached_metric:  -304.87603<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269439<br />detached_metric:  -236.72123<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269440<br />detached_metric:  -363.54061<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269441<br />detached_metric:   410.24545<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 269442<br />detached_metric:   658.27350<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 1","mapobject_id: 267101<br />detached_metric:  -134.52048<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267102<br />detached_metric:  -407.49534<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267103<br />detached_metric:  -323.91036<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267104<br />detached_metric:  -363.57815<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267105<br />detached_metric:  -855.19603<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267107<br />detached_metric:  -301.03982<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267108<br />detached_metric:  -169.70084<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267109<br />detached_metric:  -228.63926<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267110<br />detached_metric:   474.48795<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267111<br />detached_metric:  -309.00000<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267112<br />detached_metric:   354.89677<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267113<br />detached_metric:   847.72449<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267114<br />detached_metric:  -329.00000<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 1","mapobject_id: 267697<br />detached_metric:   448.46048<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 267698<br />detached_metric:   404.39884<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 267699<br />detached_metric:   428.71924<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 267700<br />detached_metric:   497.99467<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 267701<br />detached_metric:   344.19178<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 267702<br />detached_metric:   476.47213<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 1","mapobject_id: 280050<br />detached_metric:   451.61139<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280051<br />detached_metric:   537.18182<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280053<br />detached_metric:   567.82322<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280054<br />detached_metric:   554.67921<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280055<br />detached_metric:   452.46260<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280056<br />detached_metric:  -250.88486<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280058<br />detached_metric:   120.60474<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280059<br />detached_metric:  -365.51884<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280060<br />detached_metric:   411.99423<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280062<br />detached_metric:  -282.54630<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280063<br />detached_metric:  -228.12596<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280064<br />detached_metric:  -533.39010<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280066<br />detached_metric:  -432.57083<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280067<br />detached_metric:   442.57143<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 280068<br />detached_metric:  -166.40364<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 2","mapobject_id: 264086<br />detached_metric:  -705.38900<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264088<br />detached_metric:  -298.85662<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264089<br />detached_metric:  -375.19128<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264090<br />detached_metric:   -88.00643<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264091<br />detached_metric:  -358.86828<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264092<br />detached_metric:  -355.26254<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264093<br />detached_metric:   -93.75931<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264094<br />detached_metric:  -693.79845<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264095<br />detached_metric:  -345.28957<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264096<br />detached_metric:  -475.60817<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264097<br />detached_metric:  -600.99781<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264099<br />detached_metric:  -143.90365<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264100<br />detached_metric:  -526.82766<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264102<br />detached_metric:  -158.17778<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264103<br />detached_metric:  -419.35056<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264104<br />detached_metric:  -271.13793<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264105<br />detached_metric:  -513.16042<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 264106<br />detached_metric:  -259.52811<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 2","mapobject_id: 284542<br />detached_metric:  -176.38210<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284543<br />detached_metric:  -241.53034<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284544<br />detached_metric:  -446.70026<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284545<br />detached_metric:  -363.91498<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284546<br />detached_metric:  -385.03293<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284547<br />detached_metric:  -542.28773<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284548<br />detached_metric:  -120.81932<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284549<br />detached_metric:  -122.99190<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284550<br />detached_metric:  -124.99394<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284551<br />detached_metric:  -582.32151<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284553<br />detached_metric:  -125.86503<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284554<br />detached_metric:  -446.58803<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 284556<br />detached_metric:  -481.83636<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 2","mapobject_id: 268997<br />detached_metric:  -426.07853<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269001<br />detached_metric:  -363.24846<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269002<br />detached_metric:  -309.78998<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269004<br />detached_metric:  -301.29109<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269005<br />detached_metric:  -460.49892<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269006<br />detached_metric:  -286.01059<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269007<br />detached_metric:  -414.20463<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269008<br />detached_metric:  -418.24937<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269009<br />detached_metric:  -161.86499<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269010<br />detached_metric:  -296.59227<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269011<br />detached_metric:  -889.08515<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269013<br />detached_metric:  -251.09302<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269015<br />detached_metric:  -537.23630<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269016<br />detached_metric:  -204.66038<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269017<br />detached_metric:  -120.47667<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269018<br />detached_metric:  -632.60081<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269019<br />detached_metric:  -389.31388<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269020<br />detached_metric:  -311.75426<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269022<br />detached_metric:  -228.97543<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269023<br />detached_metric:   555.04656<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 2","mapobject_id: 269226<br />detached_metric:  -194.54381<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269227<br />detached_metric:  -239.95890<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269229<br />detached_metric:   512.77342<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269230<br />detached_metric:  -444.22247<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269231<br />detached_metric:  -262.68433<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269234<br />detached_metric:  -346.26168<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269235<br />detached_metric:  -282.01029<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269237<br />detached_metric:  -410.09278<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269238<br />detached_metric:  -329.71545<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269240<br />detached_metric:  -296.01094<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269241<br />detached_metric:  -403.16624<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269242<br />detached_metric:  -339.17987<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269243<br />detached_metric:  -343.29927<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269244<br />detached_metric:  -373.00993<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269245<br />detached_metric:  -404.57669<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269246<br />detached_metric:  -499.78417<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269247<br />detached_metric:  -171.41322<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269248<br />detached_metric:  -327.68138<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269250<br />detached_metric:  -312.91471<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269251<br />detached_metric:  -345.91117<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269252<br />detached_metric:  -361.47836<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269253<br />detached_metric:  -331.81936<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269254<br />detached_metric:  -654.25098<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269256<br />detached_metric:  -335.52548<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 269257<br />detached_metric:  -469.00000<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 2","mapobject_id: 271739<br />detached_metric:  -317.68442<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271740<br />detached_metric:  -297.08129<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271742<br />detached_metric:  -175.33261<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271743<br />detached_metric:  -386.74386<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271744<br />detached_metric:  -576.34595<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271745<br />detached_metric:  -324.49158<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271747<br />detached_metric:  -438.10020<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271748<br />detached_metric:  -264.60544<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271749<br />detached_metric:  -748.45648<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271750<br />detached_metric:  -140.95737<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271752<br />detached_metric:  -692.55063<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271753<br />detached_metric:  -240.79816<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271754<br />detached_metric:  -383.79344<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271755<br />detached_metric:  -396.20771<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271756<br />detached_metric:  -263.82689<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271758<br />detached_metric:  -529.83494<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271759<br />detached_metric:   -95.66786<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271761<br />detached_metric:  -309.73874<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271762<br />detached_metric:  -239.57947<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271764<br />detached_metric:  -327.59236<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271766<br />detached_metric:  -560.94464<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271767<br />detached_metric:  -340.21387<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271769<br />detached_metric:  -306.51695<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 271770<br />detached_metric:   573.73585<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 3","mapobject_id: 268714<br />detached_metric:  -355.89732<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268715<br />detached_metric:  -395.42561<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268716<br />detached_metric:  -399.07515<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268717<br />detached_metric:  -332.25078<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268718<br />detached_metric:  -487.37968<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268719<br />detached_metric:  -344.33499<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268720<br />detached_metric:  -325.83790<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268721<br />detached_metric:  -197.38107<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268722<br />detached_metric:  -134.26667<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268724<br />detached_metric:  -309.00523<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268725<br />detached_metric:  -344.87402<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268726<br />detached_metric:  -274.92822<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268727<br />detached_metric:  -541.35537<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268728<br />detached_metric:  -247.67347<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268729<br />detached_metric:  -460.50000<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268730<br />detached_metric:  -259.37566<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268731<br />detached_metric:  -243.44094<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268732<br />detached_metric:  -145.16129<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268733<br />detached_metric:  -534.60245<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268734<br />detached_metric:  -207.38170<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268736<br />detached_metric:  -479.95283<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268737<br />detached_metric:  -173.08961<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268738<br />detached_metric:   -93.58786<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268739<br />detached_metric:  -317.34337<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268740<br />detached_metric:  -293.60895<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268741<br />detached_metric:  -417.02791<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268742<br />detached_metric:  -293.37722<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268744<br />detached_metric:  -204.10542<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 268745<br />detached_metric:  -194.56757<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 3","mapobject_id: 265695<br />detached_metric:  -242.45902<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265696<br />detached_metric:  -385.65668<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265698<br />detached_metric:  -780.05063<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265699<br />detached_metric:   411.74510<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265700<br />detached_metric:  -122.72500<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265701<br />detached_metric:  -346.64855<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265702<br />detached_metric:  -429.71002<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265703<br />detached_metric:  -667.06775<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265704<br />detached_metric:   500.74286<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265706<br />detached_metric:  -236.42907<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265707<br />detached_metric:  -396.72973<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265708<br />detached_metric:  -158.28280<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265710<br />detached_metric:  -194.07571<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 265712<br />detached_metric:  -263.25259<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 3","mapobject_id: 277570<br />detached_metric:  -149.15123<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277571<br />detached_metric:  -251.09886<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277573<br />detached_metric:  -311.35085<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277574<br />detached_metric:  -203.79487<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277575<br />detached_metric:  -199.11919<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277576<br />detached_metric:  -280.54967<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277577<br />detached_metric:  -131.24444<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277578<br />detached_metric:  -213.41514<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277580<br />detached_metric:  -233.82000<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277581<br />detached_metric:  -339.40673<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277582<br />detached_metric:  -201.60513<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277584<br />detached_metric:  -154.13402<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277585<br />detached_metric:  -232.38080<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277586<br />detached_metric:  -540.84314<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277587<br />detached_metric:  -361.66828<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277588<br />detached_metric:  -230.49537<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277589<br />detached_metric:  -169.05128<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277590<br />detached_metric:  -386.43235<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277591<br />detached_metric:  -116.93821<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277593<br />detached_metric:  -405.26812<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277594<br />detached_metric:  -258.80761<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 277599<br />detached_metric:  -174.11765<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 3","mapobject_id: 286487<br />detached_metric:  -306.44972<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286488<br />detached_metric:  -253.46956<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286489<br />detached_metric:  -248.05924<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286490<br />detached_metric:  -409.51151<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286491<br />detached_metric:   130.52000<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286492<br />detached_metric:  -257.13112<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286493<br />detached_metric:  -392.17248<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286496<br />detached_metric:  -241.44293<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286497<br />detached_metric:  -271.02835<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286499<br />detached_metric:  -158.19038<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286500<br />detached_metric:  -407.47911<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286501<br />detached_metric:  -518.13207<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286502<br />detached_metric:  -360.21663<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286504<br />detached_metric:  -355.01466<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286505<br />detached_metric:  -451.13321<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 286506<br />detached_metric:  -443.30063<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 3","mapobject_id: 278745<br />detached_metric:  -286.75806<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278746<br />detached_metric:  -214.25641<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278747<br />detached_metric:  -181.91765<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278748<br />detached_metric:  -228.39519<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278749<br />detached_metric:  -595.21824<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278750<br />detached_metric:  -344.28421<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278751<br />detached_metric:  -343.75660<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278752<br />detached_metric:  -551.68033<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278753<br />detached_metric:  -248.54046<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278754<br />detached_metric:  -228.80275<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278755<br />detached_metric:  -207.65882<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278756<br />detached_metric:  -380.63963<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278757<br />detached_metric:  -138.57250<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278759<br />detached_metric:  -297.24370<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278760<br />detached_metric:   -95.80600<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278761<br />detached_metric:   -69.68926<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278764<br />detached_metric:  -161.40533<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278765<br />detached_metric:  -424.90683<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278766<br />detached_metric:  -446.00862<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278767<br />detached_metric:  -312.32374<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 278768<br />detached_metric:  -200.95238<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 4","mapobject_id: 280123<br />detached_metric:  -586.80993<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280125<br />detached_metric:  -231.07443<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280126<br />detached_metric:  -309.70395<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280127<br />detached_metric:  -392.60894<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280128<br />detached_metric:  -262.66432<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280130<br />detached_metric:  -263.21368<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280131<br />detached_metric:  -407.80636<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280132<br />detached_metric:  -223.37986<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280133<br />detached_metric:  -268.05208<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280134<br />detached_metric:  -420.56452<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280135<br />detached_metric:   -61.59102<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280136<br />detached_metric:   -70.17337<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280137<br />detached_metric:  -807.26577<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280139<br />detached_metric:  -250.45975<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280140<br />detached_metric:  -223.20366<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280141<br />detached_metric:  -261.69168<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280142<br />detached_metric:  -245.12778<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280143<br />detached_metric:  -381.29730<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 280144<br />detached_metric:  -197.33571<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 4","mapobject_id: 264174<br />detached_metric:   263.78316<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264175<br />detached_metric:  -794.37710<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264176<br />detached_metric:  -308.00494<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264177<br />detached_metric:  -421.35429<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264178<br />detached_metric:  -310.63272<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264179<br />detached_metric:  -224.53799<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264181<br />detached_metric:  -314.11644<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264182<br />detached_metric:  -231.44343<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264184<br />detached_metric:  -429.43325<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264185<br />detached_metric:  -199.66342<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264186<br />detached_metric:  -376.23244<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264187<br />detached_metric:  -236.58351<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264188<br />detached_metric:  -202.14240<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264189<br />detached_metric:  -148.67317<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264190<br />detached_metric:  -324.87840<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264191<br />detached_metric:  -109.51413<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264192<br />detached_metric:  -406.15720<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264194<br />detached_metric:  -192.19289<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264195<br />detached_metric:  -334.70259<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264197<br />detached_metric:  -326.27877<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 264198<br />detached_metric:  -115.04730<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 4","mapobject_id: 275283<br />detached_metric:  -122.07143<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275284<br />detached_metric:  -193.70470<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275285<br />detached_metric:  -378.34237<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275288<br />detached_metric:  -173.71074<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275289<br />detached_metric:  -178.03966<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275290<br />detached_metric:  -312.27692<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275291<br />detached_metric:  -315.17043<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275292<br />detached_metric:  -418.06962<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275293<br />detached_metric:  -312.92562<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275294<br />detached_metric:  -291.37037<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275295<br />detached_metric:  -131.84000<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275296<br />detached_metric:  -283.40532<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275298<br />detached_metric:  -243.71930<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275299<br />detached_metric:  -157.43258<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275300<br />detached_metric:  -200.77105<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275301<br />detached_metric:  -255.18091<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275302<br />detached_metric:  -222.32390<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275303<br />detached_metric:  -341.48829<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275304<br />detached_metric:  -864.05136<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275305<br />detached_metric:   254.40718<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275306<br />detached_metric:  -270.00908<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275307<br />detached_metric:  -272.91885<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275308<br />detached_metric:  -660.50948<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275309<br />detached_metric:  -102.50162<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275310<br />detached_metric:  -233.97526<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 275311<br />detached_metric:  -453.89286<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 4","mapobject_id: 282949<br />detached_metric:  -251.11497<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282952<br />detached_metric:  -195.76582<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282954<br />detached_metric:  -328.01392<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282955<br />detached_metric:  -527.91778<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282957<br />detached_metric:  -228.97847<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282958<br />detached_metric:  -380.37953<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282959<br />detached_metric:  -237.84401<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282960<br />detached_metric:  -142.38523<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282961<br />detached_metric:  -473.49198<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282962<br />detached_metric:  -154.05682<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282963<br />detached_metric:  -190.57313<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282965<br />detached_metric:  -120.54424<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282967<br />detached_metric:  -357.88487<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282968<br />detached_metric:  -275.15901<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282970<br />detached_metric:  -126.19938<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282972<br />detached_metric:  -125.36218<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282973<br />detached_metric:  -228.20049<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282974<br />detached_metric:  -558.77621<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282975<br />detached_metric:  -158.76136<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282976<br />detached_metric:  -436.68173<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282977<br />detached_metric:  -226.62419<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282978<br />detached_metric:  -309.62252<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282979<br />detached_metric:  -321.64021<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282981<br />detached_metric:  -398.40000<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282982<br />detached_metric:  -267.24934<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 282984<br />detached_metric:  -508.85156<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 4","mapobject_id: 266619<br />detached_metric:  -584.34584<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266620<br />detached_metric:  -191.89955<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266621<br />detached_metric:  -250.46182<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266622<br />detached_metric:  -267.47911<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266623<br />detached_metric:   259.80892<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266624<br />detached_metric:  -489.40043<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266626<br />detached_metric:  -274.98374<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266627<br />detached_metric:  -220.00721<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266628<br />detached_metric:  -285.34584<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266629<br />detached_metric:  -176.82034<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266632<br />detached_metric:  -289.65964<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266633<br />detached_metric:  -177.30612<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266634<br />detached_metric:  -240.27613<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266635<br />detached_metric:  -408.46178<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266637<br />detached_metric:  -289.90710<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266638<br />detached_metric:   -71.73575<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266639<br />detached_metric:  -334.41667<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266640<br />detached_metric:   -94.96042<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266641<br />detached_metric:  -314.37716<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266642<br />detached_metric:  -154.74631<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266643<br />detached_metric:  -341.12706<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266644<br />detached_metric:  -497.71842<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266645<br />detached_metric:  -361.50160<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266646<br />detached_metric:  -182.46154<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266647<br />detached_metric:  -230.72868<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266648<br />detached_metric:  -243.98952<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266649<br />detached_metric:  -210.88754<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266650<br />detached_metric:  -196.98355<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266651<br />detached_metric:  -150.43396<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266652<br />detached_metric:  -225.76202<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266653<br />detached_metric:  -336.52147<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266654<br />detached_metric:  -102.44282<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 266657<br />detached_metric:  -165.75352<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 5","mapobject_id: 269348<br />detached_metric:  -274.80267<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269349<br />detached_metric:  -348.68893<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269350<br />detached_metric:  -143.35129<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269351<br />detached_metric:  -452.67347<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269352<br />detached_metric:  -357.29834<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269353<br />detached_metric:  -503.71271<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269354<br />detached_metric:  -313.49176<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269355<br />detached_metric:  -396.65839<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269356<br />detached_metric:  -252.80367<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269357<br />detached_metric:  -311.87151<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269359<br />detached_metric:  -287.87798<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269360<br />detached_metric:  -339.36469<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269361<br />detached_metric:  -377.45541<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269362<br />detached_metric:  -447.53333<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269363<br />detached_metric:  -415.89691<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269364<br />detached_metric:  -278.02636<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269365<br />detached_metric:  -140.51748<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 269366<br />detached_metric:  -181.45338<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 5","mapobject_id: 285096<br />detached_metric:  -369.47929<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285097<br />detached_metric:  -736.17610<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285098<br />detached_metric:  -249.82042<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285099<br />detached_metric:  -205.30015<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285100<br />detached_metric:  -362.41348<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285101<br />detached_metric:  -307.17249<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285102<br />detached_metric:  -621.23322<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285104<br />detached_metric:  -277.05693<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285105<br />detached_metric:  -404.78054<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285106<br />detached_metric:  -186.81474<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285107<br />detached_metric:  -434.92240<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285108<br />detached_metric:  -439.09231<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285109<br />detached_metric:  -280.94389<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285110<br />detached_metric:  -361.46012<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285111<br />detached_metric:  -141.33590<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285112<br />detached_metric:  -439.05206<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285113<br />detached_metric:  -423.76066<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 285114<br />detached_metric:  -302.08268<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 5","mapobject_id: 276241<br />detached_metric:  -637.08867<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276242<br />detached_metric:  -447.76871<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276243<br />detached_metric:  -281.60328<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276245<br />detached_metric:  -286.06977<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276247<br />detached_metric:  -364.89753<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276248<br />detached_metric:  -446.87703<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276249<br />detached_metric:  -558.41577<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276250<br />detached_metric:  -369.59050<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276251<br />detached_metric:  -245.22284<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276252<br />detached_metric:  -392.81551<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276253<br />detached_metric:  -312.63286<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276254<br />detached_metric:  -607.86348<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276255<br />detached_metric:  -794.92945<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276256<br />detached_metric:  -182.71746<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276257<br />detached_metric:  -277.25840<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276258<br />detached_metric:  -379.96711<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 276259<br />detached_metric:  -329.81663<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 5","mapobject_id: 284812<br />detached_metric:  -296.41912<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284813<br />detached_metric:  -362.38484<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284814<br />detached_metric:  -375.07088<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284815<br />detached_metric:  -312.61905<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284816<br />detached_metric:  -395.33542<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284817<br />detached_metric:  -302.99104<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284818<br />detached_metric: -1033.49550<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284820<br />detached_metric:  -100.85886<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284821<br />detached_metric:  -248.66795<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284822<br />detached_metric:  -235.24716<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284823<br />detached_metric:  -299.25390<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284824<br />detached_metric:  -351.44148<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284825<br />detached_metric:  -226.40850<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284826<br />detached_metric:  -198.93264<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284827<br />detached_metric:  -519.82650<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 284829<br />detached_metric:  -235.88034<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 5","mapobject_id: 269539<br />detached_metric:  -120.10575<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269540<br />detached_metric:  -288.01792<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269541<br />detached_metric:  -347.66855<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269542<br />detached_metric:  -429.51954<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269544<br />detached_metric:  -272.17479<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269545<br />detached_metric:  -349.22353<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269546<br />detached_metric:  -156.90664<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269547<br />detached_metric:  -338.70000<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269548<br />detached_metric:  -240.98000<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269549<br />detached_metric:  -839.42462<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269550<br />detached_metric:  -455.73064<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269551<br />detached_metric:  -525.44621<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269552<br />detached_metric:  -591.58065<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269553<br />detached_metric:  -139.84241<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269554<br />detached_metric:  -194.31610<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269555<br />detached_metric:  -375.74688<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269556<br />detached_metric:  -303.90438<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269558<br />detached_metric:  -218.28688<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269559<br />detached_metric:  -196.18526<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269560<br />detached_metric:  -402.96308<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269561<br />detached_metric:  -216.51789<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269562<br />detached_metric:  -308.41030<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 269563<br />detached_metric:  -228.14733<br />well_name: C10 <br>well_pos_x: 0 <br>well_pos_y: 6","mapobject_id: 276149<br />detached_metric:  -249.85080<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276150<br />detached_metric:  -272.60414<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276151<br />detached_metric:  -513.79727<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276153<br />detached_metric:  -376.19776<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276155<br />detached_metric:  -528.27115<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276157<br />detached_metric:  -452.16106<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276158<br />detached_metric:  -277.09880<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276159<br />detached_metric:  -502.11364<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276160<br />detached_metric:  -242.18147<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276161<br />detached_metric:  -323.37589<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276162<br />detached_metric:  -158.75291<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276165<br />detached_metric:  -195.50270<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276166<br />detached_metric:   -97.85141<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276167<br />detached_metric:  -117.20489<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276168<br />detached_metric:  -416.74864<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276169<br />detached_metric:  -106.67196<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276170<br />detached_metric:  -294.81356<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276171<br />detached_metric:  -451.51724<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 276172<br />detached_metric:  -253.40206<br />well_name: C10 <br>well_pos_x: 1 <br>well_pos_y: 6","mapobject_id: 280908<br />detached_metric:  -561.69964<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280909<br />detached_metric:  -555.30995<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280912<br />detached_metric:  -104.66279<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280913<br />detached_metric:  -472.78793<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280914<br />detached_metric:  -253.22121<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280916<br />detached_metric:  -756.03750<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280917<br />detached_metric:  -257.11518<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280918<br />detached_metric:  -471.60870<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280919<br />detached_metric:  -428.45627<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280920<br />detached_metric:  -327.34165<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280921<br />detached_metric:  -264.70699<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280922<br />detached_metric:  -183.45435<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280923<br />detached_metric:  -292.50198<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280924<br />detached_metric:  -284.63580<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280925<br />detached_metric:  -282.60851<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280927<br />detached_metric:  -468.51881<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280928<br />detached_metric:  -301.70000<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280930<br />detached_metric:  -701.92731<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280931<br />detached_metric:  -226.60000<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280932<br />detached_metric:  -355.32880<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280933<br />detached_metric:  -316.28516<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280934<br />detached_metric:  -326.55363<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280935<br />detached_metric:   -92.30159<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280936<br />detached_metric:  -116.26087<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280939<br />detached_metric:  -173.66667<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280941<br />detached_metric:  -232.71429<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280942<br />detached_metric:  -458.96570<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280943<br />detached_metric:  -195.95539<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 280944<br />detached_metric:  -356.89046<br />well_name: C10 <br>well_pos_x: 2 <br>well_pos_y: 6","mapobject_id: 282478<br />detached_metric:   162.27273<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282480<br />detached_metric:  -200.32885<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282481<br />detached_metric:  -626.57565<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282482<br />detached_metric:  -473.16876<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282483<br />detached_metric:  -453.17769<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282484<br />detached_metric:  -542.67677<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282485<br />detached_metric:  -152.56534<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282487<br />detached_metric:  -346.91045<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282488<br />detached_metric:  -325.34292<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282489<br />detached_metric:  -477.67267<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282490<br />detached_metric:  -375.10646<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282491<br />detached_metric:  -298.90836<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282494<br />detached_metric:  -292.33005<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282495<br />detached_metric:  -572.74775<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 282497<br />detached_metric:  -394.41433<br />well_name: C10 <br>well_pos_x: 3 <br>well_pos_y: 6","mapobject_id: 285386<br />detached_metric:  -301.23192<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285387<br />detached_metric:  -659.30505<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285388<br />detached_metric:  -638.51179<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285390<br />detached_metric:  -418.64120<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285391<br />detached_metric:  -317.19093<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285392<br />detached_metric:  -562.01684<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285393<br />detached_metric:  -237.00373<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285394<br />detached_metric:  -221.75836<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285395<br />detached_metric:  -399.35640<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285396<br />detached_metric:  -258.20076<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285397<br />detached_metric:  -318.78997<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285398<br />detached_metric:  -180.59710<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285399<br />detached_metric:  -319.94128<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285400<br />detached_metric:  -258.24274<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285401<br />detached_metric:  -340.56010<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285402<br />detached_metric:  -785.54361<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285403<br />detached_metric:  -176.45995<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285404<br />detached_metric:  -161.93934<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285405<br />detached_metric:  -321.33721<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285406<br />detached_metric:  -193.57988<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285407<br />detached_metric:  -341.41918<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285408<br />detached_metric:  -173.93766<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285409<br />detached_metric:  -228.56200<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285410<br />detached_metric:  -288.20853<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6","mapobject_id: 285411<br />detached_metric:  -449.38391<br />well_name: C10 <br>well_pos_x: 4 <br>well_pos_y: 6"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[262965,287627],"y":[-1000,-1000],"text":"yintercept: -1000","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[262965,287627],"y":[25,25],"text":"yintercept: 25","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":54.7945205479452},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[262965,287627],"tickmode":"array","ticktext":["265000","270000","275000","280000","285000"],"tickvals":[265000,270000,275000,280000,285000],"categoryorder":"array","categoryarray":["265000","270000","275000","280000","285000"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"mapobject_id","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-1127.55649425,941.78548925],"tickmode":"array","ticktext":["-1000","-500","0","500"],"tickvals":[-1000,-500,0,500],"categoryorder":"array","categoryarray":["-1000","-500","0","500"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"detached_metric","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"1908281b4a4a":{"x":{},"y":{},"text":{},"type":"scatter"},"19083bbb2d8c":{"yintercept":{}},"190826e341b":{"yintercept":{}}},"cur_data":"1908281b4a4a","visdat":{"1908281b4a4a":["function (y) ","x"],"19083bbb2d8c":["function (y) ","x"],"190826e341b":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-8db415bae83108bb813f">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>