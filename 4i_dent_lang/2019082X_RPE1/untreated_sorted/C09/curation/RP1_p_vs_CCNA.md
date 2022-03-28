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
<div id="htmlwidget-99e7de0bf63103d7be1b" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-99e7de0bf63103d7be1b">{"x":{"data":[{"x":[7.25682370465405,6.70217693897155,8.08123499886482,7.57533229317628,6.94403478551041,6.84085623290739,7.25723629517773,6.65083323194882,6.84745130938567,6.15770473999819,6.68994367196731,6.53232263284143,6.51283184441986,7.48590658035685,7.61709061462318,6.80750557277375,6.86733702711898,6.32286819125432,6.78528898070402,8.2145336340904,7.12268335058611,6.48328122017811,6.71879343836652,6.72722232070111,7.6221707014175,8.00355575758596,6.8661960960559,6.83745642674343,7.95870533434386,6.23793032995482,7.29940444054006,6.29980303714122,6.55571883733713,6.45687440627456,7.32273534541376,7.32268260739565,6.00307066213967,6.70712603171586,6.19892903785418,6.69492483767116,7.57587496431545,7.08887496208656,8.51983908465184,6.87740662471344,7.35630061906136,6.2792828860908,7.55694678145371,7.15265738984529,8.29181645844324,7.78167925475927,7.68610379453277,7.11165230777491,7.37586201097243,6.86404915023136,7.50118747961445,6.63397429849975,7.95071239071788,6.66817273390671,6.67730322575058,7.66990969231455,6.29629662785458,6.28887273522683,6.28416518804462,8.02229501058028,7.65165958368553,6.7824435703026,6.81575686863035,6.46592264732064,7.65454107960239,7.02525758707335,6.87331593151852,6.69520246545711,7.20862650345481,6.65740271837672,7.55556972728546,6.94170805977148,7.04706767591073,6.80655695546573,6.90125670549756,6.82388559991644,7.57114468013238,6.71210920149572,7.13555520574078,5.57675079845106,6.6059005679994,6.68674023275993,6.53889641841752,6.67694638948162,6.85332117178141,6.55247144820253,6.8774329450954,7.08393190888559,7.61254330703073,6.98171345617905,7.03362281748194,6.86193623888267,7.17081964575902,7.30255271133033,7.22377443424627,6.93374952987,7.03122769779285,8.75297208547042,7.0750385882549,8.11994651425974,7.88507873717084,7.31805855230045,6.82092304793168,5.99687539314855,7.40222223343606,6.86474893360818,7.45100499781799,8.01674351535248,7.25783344405542,7.97192597838437,6.69324356058794,6.46320995410134,7.66531444716474,6.66537771920139,8.32465021706664,6.91163519104551,7.13757317718841,7.17020599904671,7.50316563549774,7.16359162492505,7.03191062812999,7.40598201578752,7.12556210820382,8.45807506494482,6.97554807403615,8.28572941920829,7.65543035173691,7.21212240573537,7.01525230163081,7.23180403657906,6.669157631409,7.32665180682553,6.44139709324046,8.28689260818833,7.67188010531758,6.32375467041228,8.67009108700718,7.08059274273935,6.97675105594241,6.40286194138837,7.66240013212438,6.50775235751674,7.29321225769135,6.88667382946758,7.79948240608741,7.10681393315328,7.54213389060928,8.16906855786922,7.55710179344524,7.88644128265816,7.20534967647946,6.79457698429381,7.58762972579056,7.04594790647232,6.48609293763525,6.26190834500985,8.03803440095035,6.84903433463812,7.16800120798332,7.78156171603401,7.35929972947325,6.60941795815431,6.66820605601958,8.55641810698763,7.87395567714069,8.27402223602789,7.56388916391151,7.7384893346558,7.83644175531923,8.50096040468062,6.87415842959737,7.31937848569301,5.70680287627369,8.31006586705108,6.86497598041005,6.19413605672772,8.29280744240404,7.87903270281766,7.9386149897063,7.09790397227135,6.30303930462526,7.7450925039909,7.01842195560819,6.87779536902833,9.24803146441846,7.37512380915743,7.45782694148737,8.17554329333051,6.72720015218774,6.84805225889655,7.24679564546114,6.49647880766547,6.51549481911702,7.6022811930581,6.21804574439409,6.79090164265781,6.7891020695231,6.78371002048823,7.49386706278911,8.22065518732047,6.75115877124952,8.4896047244264,8.31252069354172,8.32318463083649,7.19706032303727,7.2868509650653,6.52611644674282,7.03590329487947,8.48415653140898,7.11933868280358,5.97046249570902,7.87680042977406,7.32746150329377,7.53523295065226,6.57452336808138,7.28399628432323,7.97806755162794,9.18056007865239,7.12154902687588,7.07438978834932,7.95466420289217,8.41428904800478,6.35289227104734,6.90391997575258,7.42125882746776,8.6397492509812,8.58750312132358,7.93269677079804,6.67099557356834,7.75734098190558,6.98983666742836,7.39319579954436,6.78652169003571,7.89679717052848,6.851790223214,6.33535912425149,8.03056890826628,6.53065192746537,7.66310682409868,9.07547914948802,7.04676543558901,6.64228925039318,7.86163013197072,7.84937382266743,7.30999039374496,7.07425455062227,6.50355561476145,7.57386402227261,7.45613584000773,7.23906266841335,6.1597743198415,7.45672487216409,7.73838947148375,6.76127947490751,6.65080723055651,6.86856595134331,7.58849738307746,7.93054962126595,8.41433028705257,8.23038324770477,7.68181376541907,8.23348858206829,8.49941354783312,6.56292770188632,7.69456604517512,7.91709954401015,7.22362743206415,8.11231858119281,6.78863340936049,8.58570684075237,8.50160119243116,7.32454305191092,6.32652174474497,6.65854810564923,9.20792655219011,6.17437699463373,7.44615604017428,7.22788448350178,6.90348771629094,8.09840486615584,7.13889450038637,7.32076010322214,6.94855056486394,7.89497727534103,6.67994918766314,7.23093172928762,6.97784196127547,6.8153933642206,6.58611557967892,6.460461499157,6.54154999574117,7.60582522711598,8.49194184739716,8.51224459213624,7.17544533850846,8.30989594540715,7.13947747633574,6.93418229120865,7.66524204656435,7.677794321636,6.90126917323672,7.60001243676319,7.39147051835878,6.99171300609255,6.66215198162634,7.13490318303379,7.6914411111589,7.78506989154674,6.90285160454465,7.28749994380021,7.32110331387075,7.2590547479614,7.0973203744579,7.35908398404099,7.2188496496216,7.30491469900982,7.39552575559684,7.28972263538232,7.42576995127744,6.93649367406281,7.22767811633488,6.88284512255113,7.13016470816925,6.74969767446644,7.24192780442068,6.98579125113455,7.31993771494285,10.4174862590351,6.93440857848783,7.03940331341821,9.18701823008436,7.86230824918769,7.29592622895721,7.45418503340657,7.9734359271378,7.21502278033402,8.56251604469814,6.88440509293751,9.92272400959378,10.3824594483703,7.27806798803033,7.759214075843,8.36850646150769,7.48241340079413,7.21998181626075,7.13139358822077,7.11335344018168,6.66097637780643,7.16154238156672,7.14588121129242,7.05330563093219,7.31564001331379,6.97569778236961,7.15556006081065,7.37283470009298,7.54948323927651,7.14210042855046,7.94085174462581,7.08043314138107,7.35843363530629,9.28417302552289,6.81795039756209,7.11150363837363,7.90637050047819,7.52646011467275,8.63897089354397,6.85991973116078,7.41790419787722,7.6108942404131,8.58702549907514,7.81998910146912,7.40148142670053,7.19118357960353,7.84793503339789,7.11045984021998,8.40049723367024,7.47125474772764,7.72707845759469,6.92145168411527,7.72275579127075,7.14367224448549,6.67992097603913,6.81750089183057,7.16055404414548,7.39931353582945,6.45537595051406,7.86092754920043,7.81904841057122,7.11303749005413,6.83436618610433,6.89538756380528,7.39161082727396,8.08667613522635,6.95677766867616,6.52005151627865,8.27980781868492,6.56327244602954,8.81894674502585,7.81808805529603,8.25806418552363,6.5018371849023,8.82727785216554,6.15852842041596,7.50970048096946,8.05057879294009,6.72666114006788,6.82630283960382,7.47789210510455,7.83073941019154,7.09381367340273,6.53558119555381,7.05683805835441,6.9377175305831,7.80796053362771,7.97868394415821,6.79550915780159,6.73472729821169,6.53787843299747,7.07903728162989,7.61479618241348,6.99065360964743,7.47926305608433,6.84988638817551,7.52570416261272,8.60664722091745,7.26685662211999,6.60765736153751,7.35908734968197,7.41852657687479,7.65903367030414,6.96502642837449,8.02709098088502,7.27368097149077,6.93463414625815,5.548682324332,6.90756208599076,7.45106931679579,7.610084244729,6.56957539455663,6.41702322470867,7.73063317713603,7.47569633377483,7.67051872805926,7.11497830201149,7.14629366816553,7.06444106850287,6.76000753425937,7.16332675793692,7.98028466261245,7.05560476047846,6.65033126916243,7.6953156371622,7.08546519323821,7.56818619638263,7.81374325104458,9.15336925004768,7.28824227067523,7.54859278489968,8.01177500089149,6.54185995342742,7.14612458060967,6.55909142201264,6.45160739458306,6.48812415648714,6.74571379465744,7.68073625246241,8.49709814623555,7.22897864765415,7.64323462222117,8.02087443918698,7.1967405104232,9.45407967465736,6.73472702728634,7.05999247069358,7.89413458852573,8.92377815693104,6.6191517699298,6.62590741728332,6.51790555352995,6.62246628940728,7.04748936674786,9.35974625331895,6.30665430795631,7.23593268091646,6.13843120212717,8.16229657104618,6.89993128822947,6.84468045936606,7.33364810866908,7.29930371856414,7.66651435213945,7.46271071725821,7.36208924619001,7.05161540547074,6.74097808613191,7.96523165548561,7.44457957893936,7.54177049281131,6.71993843952182,7.34243050074593,7.9147529680093,6.8801957289431,7.58209110905726,7.44246338821609,6.93624139658175,7.76295165540394,7.02201608549619,8.03890530830944,8.29383650912937,7.29583219584229,8.728307317167,7.4303560509577,7.24264484112902,6.41351201799452,7.28217110027412,7.19231477286202,6.38943707694693,6.72645168329136,6.9110982054478,6.58967255179364,7.8503739538427,7.51324186870293,6.75768747225035,6.78285784309413,6.01585933961733,7.05857198629536,8.34569536225911,9.28343729944566,7.05567683393609,8.40412395759003,7.20143458495589,7.30018374712359,6.84898599073093,7.22696449234009,7.61702415278612,7.74879116882722,6.98434387537882,7.2995793248851,7.1993619749917,7.8338192849959,9.50118847104274,7.24960282859462,7.89881070432165,7.818432998616,7.29452235741043,6.57681498556878,6.81589017716137,8.14733666980712,8.01078884438199,7.48367507942814,7.01571069791034,6.88358555486316,7.10197160671799,7.02015145248044,7.41606925457686,7.3309941274374,7.08728987720639,7.1527561842892,6.6733208503082,7.03491168918088,7.5058992163128,6.72716594555775,7.16347988333294,6.83274510462229,7.63441862730765,6.67251079790559,8.51062980413453,6.83182183466072,7.00374175491399,6.48305781672548,7.86573901687973,7.98146923809676,6.96631551411659,7.62965154091931,8.09144583293182,7.28573673735752,8.58669332352996,8.77810988392771,6.76306672861327,7.29529986866128,7.94844259436975,6.84175732732298,8.66038149164697,7.05195679462463,7.27186141818353,7.33244635432911,8.01114268752957,7.98424042523012,8.0692344198907,7.55637315200497,6.46228372711113,8.00117302015679,9.108678471934,7.6047808252241,8.19184435515787,7.97609595270359,6.90400768903578,7.01691338335671,7.30861875842645,9.04890979002251,8.76210827775775,7.05574655068147,6.94683881263442,8.1581183814211,7.27785124535319,7.89158474337231,7.90775567670101,6.59166896106845,9.81336170031523,7.15876114431045,7.25326490140655,7.44597210514875,7.21136624130836,7.73984422194164,6.51594115023581,7.08910838875572,7.5286632424324,7.56644722747296,7.09656762577828,6.525647917261,7.70966241543962,7.67465000839363,7.31176021066326,7.37043950347807,9.01598314491904,6.58367209431777,6.66960551378168,6.83875037041151,6.83508363391695,7.80248468687381,6.53365977383155,6.44230759152648,7.29036680329288,6.57219981344945,6.85379499285483,7.11826358392117,6.80778339037752,6.77437216254095,8.13815350713051,7.57044754614367,6.94454613644892,6.42909646699268,8.09005700322753,7.04259354016729,5.99729240763651,7.49682213958754,6.96401005267586,6.82462472193943,7.20862581095416,7.06221482465911,7.4246255933179,7.05596515681909,7.38658105035572,8.06981493720929,7.90166418666851,7.35173362482657,6.85839549897068,7.38301127823802,7.64894405450328,8.30529175949213,6.8633013217503,7.08708393553633,7.10875960735904,7.05362017694492,7.92751558694652,7.32977617846243,7.08799103613429,7.41168714282089,7.28667857792715,7.05528243550119,7.38051129987305,7.00601298043012,8.80387495397874,6.44123156029948,6.8189625259202,7.11947597469042,7.0232457808831,9.06070165652786,6.75137117972281,7.8420849008258,7.34693898850627,7.00612760838932,7.72307839094298,6.92263438554493,7.11928611580949,8.03366449508206,6.8789510575388,6.70988442572534,7.53571972825197,7.79528907589992,7.82027192387974,6.67931449676074,6.75394646464672,7.58198017240697,7.84057024866607,6.24680382396,8.20498541699865,8.9196714506482,6.65883967536559,6.62080680047251,6.70128292114262,6.46992080626227,6.76717873361254,9.33332393894685,6.24356148788087,7.55226776116165,6.49197301136373,7.19140411687198,7.62899129510278,7.33705852104848,8.2177963153425,8.12347238749936,7.99622330951303,6.60444022558757,7.71564658710505,8.0794795677148,8.61762653684033,8.39241178740569,6.61045365745506,7.03787306775938,7.39595010289844,7.28445020699383,6.80856743552904,8.94478482783858,7.56528568794894,6.79085667904764,7.61338001610918,7.34865051213635,6.71703230958067,7.80811314884593,7.04542358485317,7.5081141142137,7.47336611830647,7.50187284390316,6.6651543962521,7.77694399487125,7.00749454028448,6.48912290728831,6.76599280887264,6.45997798509589,7.28009410346516,7.18592417269305,7.21259642803613],"y":[9.59234525023449,8.60708489044931,9.48027401565701,7.01988659782781,9.28567537100725,8.02457561029402,8.67023893303886,8.29844116647195,8.214229714656,7.36532742603428,6.62383412349149,8.59805674074802,8.74290437078058,9.64782312855855,8.87088582062359,8.97842613866867,9.13275779537079,8.53422529197596,9.13840780649243,9.30157552959662,8.54229546337958,9.1303486538372,8.98441845880114,9.35459616679601,8.71499811960168,9.34107124149684,10.0712872803537,8.92898321383742,9.96195226090599,9.48721287372791,10.0925304693814,8.31682481515221,8.57456940716696,8.97075955233861,9.45233125932146,8.37713088708507,9.26476245365606,4.53168525230624,8.75912766421201,9.78719267346481,8.87817270109908,9.0080428302564,9.92417851952499,7.16859512554769,9.11610762843359,9.09050500430319,9.7561041444006,9.04052541594615,9.31048717084444,8.83247674940285,9.36256457565543,9.14251406969545,9.50697357455602,10.1536136812212,8.80419790968801,9.53407007320123,9.55555304777928,9.6115867042939,9.42418265186906,8.77338304609144,8.88850625307638,9.08380210121826,8.91685251128042,10.1629619402658,8.22193658328892,9.27343613225347,8.61144751881299,9.20845280461089,9.59181842116391,6.54106487183639,8.47572498955471,8.67300232227603,8.46054599794141,4.94235643360365,8.78552734249688,9.84880334270226,9.42738304939457,9.73402781960502,9.04310855358192,9.34261091965961,10.0356758408693,8.97846001467905,8.9016030374396,8.24309767574297,8.83926540056951,9.44713881436955,8.77274525597132,8.78051616580506,5.03347225056113,9.66639205588953,4.7476920199952,8.52507300693125,9.49831492958522,8.89021236642981,8.90959069799463,8.22925757812443,8.24024572812602,8.83480642416561,4.65975379894733,8.55491210576902,9.08505677565748,5.39937955161098,8.59888262815304,9.06309640171302,9.44282903957491,9.67609340526968,8.20760531838181,4.62148839928838,9.14554146764721,9.15683910577216,9.14782133036604,9.26324190042692,9.04120281827053,9.52251807873889,8.84403567113302,8.67777771467256,7.42410918468426,8.78067484792679,8.88038353344601,8.94333531775511,8.59032927660083,9.63030925292731,10.0405955550264,9.85721543610573,8.15633924381484,8.78528102820173,9.22132133640788,9.91096388884982,9.94411615544436,9.6183742066097,9.53531901456931,9.37607996700666,8.48468490440322,8.72815132513986,4.94592807923841,8.52933397429148,9.31354790258255,9.12027204016344,9.10060759176282,5.96531631391619,8.59267978937778,8.56501194091021,8.8513496344064,8.01049855516133,9.39936206733691,8.45915835369566,4.53565442508069,8.72051019935182,8.97362732897101,9.23346894559815,9.49440851171557,9.57452595275744,10.1503850052656,10.2667145083653,9.70221405184458,9.20189607313561,11.1535609728431,8.68297188650525,8.85642910501075,9.32250047878635,9.82056883955805,8.74229937002954,8.37627207861079,9.13277076375276,8.99229568265578,9.24693470450517,4.40595615574225,9.53980397617698,10.1216632763546,9.32196118407119,9.11483739637387,6.15601675884028,9.80630472164071,9.01994918381135,9.59704628002368,9.36182250770888,7.34520026965907,10.5800061944121,9.12043402905322,8.8421918582734,10.5917687197067,9.65568766338231,9.1337264100041,10.0467091308504,8.73138165960332,9.6505641878423,10.4639759407276,10.4777034642373,10.4123278908984,10.1581684627826,9.902415168692,9.47469621029728,9.86933271262646,9.0350648808142,9.71443886534731,8.50708371976918,9.25474385856349,9.83821476679238,6.21025748296825,9.79988760750174,8.62513804383214,4.6819258494314,9.6575793349332,9.76579940458035,9.25322786034846,9.94332510444152,10.1561512164158,5.07145313785996,9.01029650082593,9.7020618066137,4.50934475186766,11.4797285064161,10.4295328091394,8.74522033047098,7.20947992851945,9.90614300175479,9.21090892939872,9.7174240612964,8.17630483858483,9.3940532691225,8.95790075707278,9.38483755287479,8.61574525758414,7.90581602490342,9.79784587267055,9.59409836115211,9.18927812359567,4.66973727661636,9.37083244920865,9.95950410335993,9.81064412432936,9.83864696731948,9.24727575452139,9.14796140613059,9.92039601055751,9.10463922707877,9.2892614153061,9.18899680986942,4.61075716177855,8.7598765451021,8.99864064537679,4.65570873698414,8.5540126331911,9.75781683393373,9.28525688544899,9.55424171705966,9.44237683242944,9.12208331507651,8.75852559590573,5.80335802017093,4.75683158582635,9.19176037655945,9.78782062863321,8.10573483533899,9.46000950512549,8.73301532168596,9.28257554243466,9.37833324994927,9.81290746782516,9.16948561636717,9.59154402703817,9.04728623753666,8.67224918451759,9.92200504545613,9.6562288275698,9.97127824754092,9.12781353828534,7.97463700583262,9.22738613110509,9.47119184074527,8.52538497403912,9.57634148551577,8.10808280427639,10.0790985458677,9.0620007535185,9.35386789131848,9.29104576319987,8.52283578017324,9.4738828534067,7.44815035137145,8.80115399889272,4.89048472951907,8.67901140084198,9.3112528599952,8.94819187507183,8.7713053176994,9.26772095218162,9.40489345786412,9.19978593051726,9.94144048543816,9.91375530839519,9.72812373944954,9.91107571832164,10.3476425449172,10.1320269764443,10.0557059399212,10.0142724023088,7.50967920144576,9.78260164909651,9.88384517072406,8.85285340762541,9.82145844042243,9.30738556247864,10.1770062662748,8.6536960016293,9.87117644796619,9.19023506825778,5.70092779363422,8.41665175210754,10.6273868698232,9.09963185001446,10.855018034076,9.26518904282657,9.4669053103668,7.35690312098049,8.84698753112027,9.13797027228324,10.1093525116134,9.79994654097073,10.2736911655228,9.14422737344508,9.60033463496366,9.89686397769201,8.8565814962273,4.80044753440724,9.60795308509294,8.61093338067801,8.31377495459716,4.72598910862863,8.712741380343,9.84023797300341,10.0685437565027,4.79430153798612,9.7438663711182,9.05888233770399,9.0772745548705,9.48158944675032,8.92261881292133,9.3511376765422,9.39668104293714,9.38493847380682,8.65422127821997,10.9362336369982,9.19411102507674,9.187491597088,9.26982630543023,9.32248440578788,9.70159352884316,8.54201831337493,8.54179419596303,9.62035206160398,5.70407654809158,5.18314747702045,4.77175355426079,4.86242317195443,8.47133807609871,9.94857616629974,9.17659600501547,4.51184454063376,9.82301788561217,9.3570720923522,8.41725472721456,9.06593839402939,8.29686447076987,9.16651939310686,8.91904808116744,6.85864363692686,9.84926169071507,9.5473251461453,9.48844228351438,5.97581204774823,8.74196245357526,10.0952398461892,8.58724616549355,8.77083272873148,9.04631150337906,5.15367966214134,8.9963808538627,4.7642749677106,10.0271425177063,9.71269499155055,10.0406214616387,9.58158777304304,9.96297900272784,10.0116951034337,7.75752995483263,9.64052906312079,4.60023757562637,9.60968151010178,8.62640883686284,8.65001588068863,9.38571106067924,9.18634753604399,8.73032439639539,8.54719939950286,10.1718135412852,9.51734902899979,5.1356018070912,8.97976751146614,9.46784611776211,4.7460284457853,9.49791807369044,9.50537009909644,10.470270546714,8.78131242186189,8.78495978501593,9.2233854659979,5.00621161222875,10.4567823504787,9.12746510165641,8.20086899577844,9.66313581129787,8.35152265909229,9.13480378202991,8.91702559711346,8.85170078527034,9.14028281284605,9.15938230639528,9.82433687972066,9.01509040808457,8.78043483322224,8.7138160802345,9.84122132920603,9.43283575850406,9.98087488615311,8.94966077815868,8.47954385281382,8.62385834453777,9.31280007052706,10.1604876016377,8.78871318171355,7.98991622822118,4.58366361087493,9.29564232979331,9.12376484455474,9.4287599574109,9.0253730024626,8.90615604079608,8.92234739773107,9.0232665040858,7.86359270344608,8.96730972807566,9.71715787977726,9.19777774567527,4.90015912230311,10.0874173253602,9.7302385333488,9.3075595230586,7.0608293450915,9.3360141496885,10.1673757989745,9.56641860758097,9.88511401398984,9.19900502503666,9.10609875372654,9.21837388744659,9.5387471693317,9.47688239804676,9.734843207263,10.7478761625643,9.27698956969831,10.1053432096143,9.14657297518197,8.61724179948883,8.80722823949842,7.75176719428636,9.45255805483744,8.86064025604201,10.0116837877748,9.06908790030853,10.7177609536096,8.72275428534965,9.60986192982521,9.7351372295698,9.27394202183424,8.69682021433539,5.18254114822662,5.77977668102652,9.57196920402185,9.96185069052166,7.38683367804996,7.71159578852943,9.03868492183406,8.8800166332014,9.201163945819,9.85221220176286,9.48365339239601,8.79335407783059,9.48241604696029,8.93061812404713,9.19143114248569,9.14706601496576,9.70111185743385,9.20402359627442,9.52016214315341,9.35412097306577,8.77624590753353,9.2986413103746,8.69799641961542,9.10151939999329,10.0309571026006,9.8182069848504,9.39784638364353,9.57068779381843,8.79087726528475,9.37905087928835,9.74929757118225,9.37518236622998,8.87757291973996,4.86638484455441,9.36019794481462,9.43896810006449,9.14583634921738,9.39907182606304,9.1763983291216,4.73804025408816,9.43893208237305,8.53524874290236,8.06775015942931,8.92139952980675,8.25884348045295,8.94514956681284,9.25319863440536,8.29392327618715,9.12885660405869,10.0259693246248,5.45388176271256,9.20268357642931,5.33158194162471,8.00194492347614,8.5720855586299,10.3398672494438,10.9111760078703,9.57239409637087,9.60166104657331,6.85608652549846,8.78323341945251,9.90779337556296,8.34268510782731,9.9773939834989,4.67306010128034,4.91971253192791,8.86721768492292,9.15504640633673,10.0419774853639,9.37661496926919,9.68198896267585,9.63370067874731,9.74686312733243,9.55213471152774,9.40350628038116,9.51061695607204,9.73473473339946,9.41757371455159,9.26951633422968,8.77620385439102,10.0402091251439,8.61463720332724,9.80604449686235,9.65625532681694,9.21340143768058,10.0012738035537,8.65475535201462,9.08997137543219,8.69947838268663,9.858786179775,9.540018598891,7.11483509605687,9.35323625437714,9.25861940437528,8.94109017915876,9.01671254581858,9.06178068440448,8.94846083142028,9.32442734789692,10.1153365707735,9.79392132061631,8.91479525074508,9.54844758166124,8.32900535415392,10.4259197709455,10.8663739944105,8.71628798877587,8.75327510442804,7.96758265416801,9.35454838009094,9.34138083829375,9.35609107085022,9.29004406161074,4.6481170372784,9.7854880062449,9.84630999943305,10.0066224717605,10.0781778076331,9.41461551548838,8.66009520135656,9.1161763987246,8.76308070201765,9.06829442386576,10.7194161454643,9.46520826386105,5.10773523311885,8.86422327654621,9.5138270674317,9.02036963453241,8.60423598290259,10.3554075718744,9.8572021135854,9.32618939999485,9.90460175764906,9.79788863518563,8.06761062500294,10.388237452877,9.86758248771614,9.28490282585826,9.1064109266847,10.7821583397326,9.86418058593339,8.56989434796923,9.83073126072307,9.60956986605953,8.96496732064629,10.0062448378484,9.7339232285807,9.01410656343291,9.70711797649514,10.1145744872507,9.45450396287425,9.91925682769196,9.48286228814326,9.20493974486436,8.92308782361568,9.30926172306245,9.85309036920262,9.33946156506946,8.64280720365052,8.74294646299282,9.4175882897798,8.41986154046997,8.83551097988824,9.12977700646539,9.34291992632628,10.315600596665,8.47763599032827,8.0858943522589,8.5444315569606,9.21487047710978,9.45245106779089,9.52107929715748,8.95791121586193,9.50671480501198,7.36666445086241,9.57692355459302,9.19519275574053,9.04411124689398,10.0922068235284,8.62759536276393,9.72243364202509,9.5909662978751,9.96295123723804,9.66915925103228,9.30301390722477,9.62314732436903,9.89393869270947,9.80758050263745,9.54200118616361,8.43052951425679,10.4051486521231,9.22393935711159,9.40822255059495,9.18617344207327,9.94255514507562,9.29236495441991,9.35123544710397,9.40105502036409,9.48763839204898,9.79000662142938,7.78404621714336,4.94753258678501,10.3388052916189,4.95374984354179,9.40415383021342,8.73067343841528,10.453359813887,9.59196301652029,9.19785150636911,8.72881168650092,8.94903858564604,9.87248004865491,9.71353122571615,8.69315503673448,8.74523733212975,9.94752214453225,9.72310123568892,10.611676179248,9.55861766596519,8.99654310927042,9.25474685551838,9.53007678362281,8.89824807203668,9.703942013663,9.41533250236477,8.9840801249472,9.79401230045496,8.99870901073755,9.11753452248984,4.80983119118932,9.95308259998184,9.39333432961559,9.08429463048132,10.0880579336325,9.55764504663558,8.5112937049372,4.452707704373,9.0295595903643,9.33243552540013,10.639457758481,9.23287422916639,9.59033753565743,10.1381239899658,10.0915369029935,5.30004323898134,9.92841703217985,9.00066933973747,10.4637840064061,4.91461589706445,7.55467488131524,9.84883796263847,9.93696467684358,9.64962850828162,10.3937049242521,8.89957776374694,7.41500440708182,9.3067431391938,8.90066497048455,9.11762066996886,9.80023087290454,9.27252527502144,9.99543398409366,8.74870095111074,4.78515247410611,8.70792823204178,9.47351319278217,8.82771319029138,4.63435733018248,8.88817275719725,8.99435343685886],"text":["Nuclei_Mean_Intensity_mean_CCNA2:  152.94018<br />Nuclei_Mean_Intensity_mean_RB1_p:  771.94018<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  104.12531<br />Nuclei_Mean_Intensity_mean_RB1_p:  389.93366<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  270.82836<br />Nuclei_Mean_Intensity_mean_RB1_p:  714.24440<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  190.72264<br />Nuclei_Mean_Intensity_mean_RB1_p:  129.77661<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  123.12968<br />Nuclei_Mean_Intensity_mean_RB1_p:  624.11816<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  114.63122<br />Nuclei_Mean_Intensity_mean_RB1_p:  260.39819<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  152.98393<br />Nuclei_Mean_Intensity_mean_RB1_p:  407.38214<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  100.48478<br />Nuclei_Mean_Intensity_mean_RB1_p:  314.83261<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  115.15644<br />Nuclei_Mean_Intensity_mean_RB1_p:  296.98160<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   71.39270<br />Nuclei_Mean_Intensity_mean_RB1_p:  164.88627<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  103.24611<br />Nuclei_Mean_Intensity_mean_RB1_p:   98.62176<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   92.56036<br />Nuclei_Mean_Intensity_mean_RB1_p:  387.50114<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   91.31828<br />Nuclei_Mean_Intensity_mean_RB1_p:  428.42664<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  179.25960<br />Nuclei_Mean_Intensity_mean_RB1_p:  802.20276<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  196.32371<br />Nuclei_Mean_Intensity_mean_RB1_p:  468.16907<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  112.01170<br />Nuclei_Mean_Intensity_mean_RB1_p:  504.40058<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  116.75472<br />Nuclei_Mean_Intensity_mean_RB1_p:  561.35040<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   80.05215<br />Nuclei_Mean_Intensity_mean_RB1_p:  370.73006<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  110.30000<br />Nuclei_Mean_Intensity_mean_RB1_p:  563.55313<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  297.04416<br />Nuclei_Mean_Intensity_mean_RB1_p:  631.03470<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  139.36103<br />Nuclei_Mean_Intensity_mean_RB1_p:  372.80967<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   89.46684<br />Nuclei_Mean_Intensity_mean_RB1_p:  560.41379<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  105.33152<br />Nuclei_Mean_Intensity_mean_RB1_p:  506.50000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  105.94872<br />Nuclei_Mean_Intensity_mean_RB1_p:  654.65734<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  197.01623<br />Nuclei_Mean_Intensity_mean_RB1_p:  420.21916<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  256.63173<br />Nuclei_Mean_Intensity_mean_RB1_p:  648.54876<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  116.66242<br />Nuclei_Mean_Intensity_mean_RB1_p: 1075.86943<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  114.36140<br />Nuclei_Mean_Intensity_mean_RB1_p:  487.40702<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  248.77632<br />Nuclei_Mean_Intensity_mean_RB1_p:  997.34737<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   75.47518<br />Nuclei_Mean_Intensity_mean_RB1_p:  717.68794<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  157.52144<br />Nuclei_Mean_Intensity_mean_RB1_p: 1091.82844<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   78.78249<br />Nuclei_Mean_Intensity_mean_RB1_p:  318.87006<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   94.07365<br />Nuclei_Mean_Intensity_mean_RB1_p:  381.24363<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   87.84416<br />Nuclei_Mean_Intensity_mean_RB1_p:  501.72727<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  160.08955<br />Nuclei_Mean_Intensity_mean_RB1_p:  700.54371<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  160.08370<br />Nuclei_Mean_Intensity_mean_RB1_p:  332.48165<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   64.13636<br />Nuclei_Mean_Intensity_mean_RB1_p:  615.13636<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  104.48312<br />Nuclei_Mean_Intensity_mean_RB1_p:   23.12987<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:   73.46214<br />Nuclei_Mean_Intensity_mean_RB1_p:  433.27154<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  103.60321<br />Nuclei_Mean_Intensity_mean_RB1_p:  883.56513<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  190.79439<br />Nuclei_Mean_Intensity_mean_RB1_p:  470.53972<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Nuclei_Mean_Intensity_mean_CCNA2:  136.13318<br />Nuclei_Mean_Intensity_mean_RB1_p:  514.86230<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  367.05160<br />Nuclei_Mean_Intensity_mean_RB1_p:  971.57295<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  117.57248<br />Nuclei_Mean_Intensity_mean_RB1_p:  143.86732<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  163.85781<br />Nuclei_Mean_Intensity_mean_RB1_p:  554.90909<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   77.66986<br />Nuclei_Mean_Intensity_mean_RB1_p:  545.14832<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  188.30752<br />Nuclei_Mean_Intensity_mean_RB1_p:  864.72893<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  142.28674<br />Nuclei_Mean_Intensity_mean_RB1_p:  526.58602<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  313.39024<br />Nuclei_Mean_Intensity_mean_RB1_p:  634.94471<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  220.04873<br />Nuclei_Mean_Intensity_mean_RB1_p:  455.86940<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  205.94336<br />Nuclei_Mean_Intensity_mean_RB1_p:  658.28320<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  138.29952<br />Nuclei_Mean_Intensity_mean_RB1_p:  565.15942<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  166.09467<br />Nuclei_Mean_Intensity_mean_RB1_p:  727.58580<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  116.48894<br />Nuclei_Mean_Intensity_mean_RB1_p: 1139.04867<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  181.16839<br />Nuclei_Mean_Intensity_mean_RB1_p:  447.02073<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   99.31738<br />Nuclei_Mean_Intensity_mean_RB1_p:  741.38035<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  247.40184<br />Nuclei_Mean_Intensity_mean_RB1_p:  752.50275<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  101.69978<br />Nuclei_Mean_Intensity_mean_RB1_p:  782.30464<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  102.34546<br />Nuclei_Mean_Intensity_mean_RB1_p:  687.00779<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  203.64459<br />Nuclei_Mean_Intensity_mean_RB1_p:  437.57395<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   78.59124<br />Nuclei_Mean_Intensity_mean_RB1_p:  473.92214<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   78.18786<br />Nuclei_Mean_Intensity_mean_RB1_p:  542.62139<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   77.93315<br />Nuclei_Mean_Intensity_mean_RB1_p:  483.32590<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  259.98688<br />Nuclei_Mean_Intensity_mean_RB1_p: 1146.45335<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  201.08471<br />Nuclei_Mean_Intensity_mean_RB1_p:  298.57231<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  110.08267<br />Nuclei_Mean_Intensity_mean_RB1_p:  618.84579<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  112.65417<br />Nuclei_Mean_Intensity_mean_RB1_p:  391.11458<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   88.39683<br />Nuclei_Mean_Intensity_mean_RB1_p:  591.58957<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  201.48674<br />Nuclei_Mean_Intensity_mean_RB1_p:  771.65835<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  130.26066<br />Nuclei_Mean_Intensity_mean_RB1_p:   93.12295<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  117.23958<br />Nuclei_Mean_Intensity_mean_RB1_p:  355.99792<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  103.62314<br />Nuclei_Mean_Intensity_mean_RB1_p:  408.16321<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  147.91520<br />Nuclei_Mean_Intensity_mean_RB1_p:  352.27200<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  100.94340<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.74663<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  188.12786<br />Nuclei_Mean_Intensity_mean_RB1_p:  441.27290<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  122.93126<br />Nuclei_Mean_Intensity_mean_RB1_p:  922.11530<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  132.24484<br />Nuclei_Mean_Intensity_mean_RB1_p:  688.53350<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  111.93807<br />Nuclei_Mean_Intensity_mean_RB1_p:  851.59745<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  119.53230<br />Nuclei_Mean_Intensity_mean_RB1_p:  527.52972<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  113.29070<br />Nuclei_Mean_Intensity_mean_RB1_p:  649.24128<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  190.16984<br />Nuclei_Mean_Intensity_mean_RB1_p: 1049.63778<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  104.84463<br />Nuclei_Mean_Intensity_mean_RB1_p:  504.41243<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  140.60998<br />Nuclei_Mean_Intensity_mean_RB1_p:  478.24399<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   47.72756<br />Nuclei_Mean_Intensity_mean_RB1_p:  302.98397<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   97.40342<br />Nuclei_Mean_Intensity_mean_RB1_p:  458.01956<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  103.01712<br />Nuclei_Mean_Intensity_mean_RB1_p:  698.02689<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   92.98309<br />Nuclei_Mean_Intensity_mean_RB1_p:  437.38055<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  102.32014<br />Nuclei_Mean_Intensity_mean_RB1_p:  439.74281<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  115.62593<br />Nuclei_Mean_Intensity_mean_RB1_p:   32.75112<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   93.86214<br />Nuclei_Mean_Intensity_mean_RB1_p:  812.59465<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  117.57463<br />Nuclei_Mean_Intensity_mean_RB1_p:   26.86567<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  135.66755<br />Nuclei_Mean_Intensity_mean_RB1_p:  368.38564<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  195.70588<br />Nuclei_Mean_Intensity_mean_RB1_p:  723.23211<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  126.38781<br />Nuclei_Mean_Intensity_mean_RB1_p:  474.48293<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  131.01815<br />Nuclei_Mean_Intensity_mean_RB1_p:  480.89919<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  116.31846<br />Nuclei_Mean_Intensity_mean_RB1_p:  300.09128<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  144.08933<br />Nuclei_Mean_Intensity_mean_RB1_p:  302.38562<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  157.86557<br />Nuclei_Mean_Intensity_mean_RB1_p:  456.60613<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  149.47645<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.27701<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  122.25499<br />Nuclei_Mean_Intensity_mean_RB1_p:  376.08426<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  130.80081<br />Nuclei_Mean_Intensity_mean_RB1_p:  543.09350<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  431.42683<br />Nuclei_Mean_Intensity_mean_RB1_p:   42.20610<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  134.83382<br />Nuclei_Mean_Intensity_mean_RB1_p:  387.72303<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  278.19381<br />Nuclei_Mean_Intensity_mean_RB1_p:  534.88925<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  236.39877<br />Nuclei_Mean_Intensity_mean_RB1_p:  695.94479<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  159.57143<br />Nuclei_Mean_Intensity_mean_RB1_p:  818.07733<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  113.05830<br />Nuclei_Mean_Intensity_mean_RB1_p:  295.62108<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   63.86154<br />Nuclei_Mean_Intensity_mean_RB1_p:   24.61538<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  169.15737<br />Nuclei_Mean_Intensity_mean_RB1_p:  566.34661<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  116.54546<br />Nuclei_Mean_Intensity_mean_RB1_p:  570.79904<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  174.97500<br />Nuclei_Mean_Intensity_mean_RB1_p:  567.24231<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  258.98837<br />Nuclei_Mean_Intensity_mean_RB1_p:  614.48837<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  153.04726<br />Nuclei_Mean_Intensity_mean_RB1_p:  526.83333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  251.06654<br />Nuclei_Mean_Intensity_mean_RB1_p:  735.46765<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  103.48254<br />Nuclei_Mean_Intensity_mean_RB1_p:  459.53651<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:   88.23077<br />Nuclei_Mean_Intensity_mean_RB1_p:  409.51648<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  202.99698<br />Nuclei_Mean_Intensity_mean_RB1_p:  171.74320<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  101.50294<br />Nuclei_Mean_Intensity_mean_RB1_p:  439.79118<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  320.60436<br />Nuclei_Mean_Intensity_mean_RB1_p:  471.26134<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  120.39529<br />Nuclei_Mean_Intensity_mean_RB1_p:  492.28000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  140.80680<br />Nuclei_Mean_Intensity_mean_RB1_p:  385.43113<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  144.02805<br />Nuclei_Mean_Intensity_mean_RB1_p:  792.52314<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  181.41697<br />Nuclei_Mean_Intensity_mean_RB1_p: 1053.22325<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  143.36923<br />Nuclei_Mean_Intensity_mean_RB1_p:  927.50769<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  130.86274<br />Nuclei_Mean_Intensity_mean_RB1_p:  285.30065<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  169.59878<br />Nuclei_Mean_Intensity_mean_RB1_p:  441.19757<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Nuclei_Mean_Intensity_mean_CCNA2:  139.63939<br />Nuclei_Mean_Intensity_mean_RB1_p:  596.89003<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  351.66917<br />Nuclei_Mean_Intensity_mean_RB1_p:  962.71429<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  125.84884<br />Nuclei_Mean_Intensity_mean_RB1_p:  985.09302<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  312.07077<br />Nuclei_Mean_Intensity_mean_RB1_p:  785.99385<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  201.61097<br />Nuclei_Mean_Intensity_mean_RB1_p:  742.02244<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  148.27406<br />Nuclei_Mean_Intensity_mean_RB1_p:  664.47908<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  129.36041<br />Nuclei_Mean_Intensity_mean_RB1_p:  358.21574<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  150.31071<br />Nuclei_Mean_Intensity_mean_RB1_p:  424.06786<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  101.76923<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.82284<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  160.52473<br />Nuclei_Mean_Intensity_mean_RB1_p:  369.47526<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   86.90680<br />Nuclei_Mean_Intensity_mean_RB1_p:  636.29320<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  312.32248<br />Nuclei_Mean_Intensity_mean_RB1_p:  556.51318<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  203.92292<br />Nuclei_Mean_Intensity_mean_RB1_p:  548.97917<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   80.10135<br />Nuclei_Mean_Intensity_mean_RB1_p:   62.47973<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  407.34040<br />Nuclei_Mean_Intensity_mean_RB1_p:  386.05960<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  135.35391<br />Nuclei_Mean_Intensity_mean_RB1_p:  378.72634<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  125.95382<br />Nuclei_Mean_Intensity_mean_RB1_p:  461.87211<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   84.61620<br />Nuclei_Mean_Intensity_mean_RB1_p:  257.86972<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  202.58733<br />Nuclei_Mean_Intensity_mean_RB1_p:  675.28938<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   90.99733<br />Nuclei_Mean_Intensity_mean_RB1_p:  351.93333<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  156.84680<br />Nuclei_Mean_Intensity_mean_RB1_p:   23.19359<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  118.33014<br />Nuclei_Mean_Intensity_mean_RB1_p:  421.82775<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  222.78100<br />Nuclei_Mean_Intensity_mean_RB1_p:  502.72559<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  137.83648<br />Nuclei_Mean_Intensity_mean_RB1_p:  601.93711<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  186.38396<br />Nuclei_Mean_Intensity_mean_RB1_p:  721.27645<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  287.82908<br />Nuclei_Mean_Intensity_mean_RB1_p:  762.46429<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  188.32775<br />Nuclei_Mean_Intensity_mean_RB1_p: 1136.50239<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  236.62214<br />Nuclei_Mean_Intensity_mean_RB1_p: 1231.93849<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  147.57962<br />Nuclei_Mean_Intensity_mean_RB1_p:  833.02388<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  111.01240<br />Nuclei_Mean_Intensity_mean_RB1_p:  588.90702<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  192.35529<br />Nuclei_Mean_Intensity_mean_RB1_p: 2278.01412<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  132.14224<br />Nuclei_Mean_Intensity_mean_RB1_p:  410.99353<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   89.64138<br />Nuclei_Mean_Intensity_mean_RB1_p:  463.50115<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   76.74008<br />Nuclei_Mean_Intensity_mean_RB1_p:  640.25397<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  262.83879<br />Nuclei_Mean_Intensity_mean_RB1_p:  904.24433<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  115.28287<br />Nuclei_Mean_Intensity_mean_RB1_p:  428.24701<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  143.80811<br />Nuclei_Mean_Intensity_mean_RB1_p:  332.28378<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  220.03081<br />Nuclei_Mean_Intensity_mean_RB1_p:  561.35545<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  164.19879<br />Nuclei_Mean_Intensity_mean_RB1_p:  509.27309<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   97.64119<br />Nuclei_Mean_Intensity_mean_RB1_p:  607.58174<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  101.70213<br />Nuclei_Mean_Intensity_mean_RB1_p:   21.19947<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  376.47705<br />Nuclei_Mean_Intensity_mean_RB1_p:  744.33279<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  234.58317<br />Nuclei_Mean_Intensity_mean_RB1_p: 1114.10020<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  309.54862<br />Nuclei_Mean_Intensity_mean_RB1_p:  640.01468<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  189.21585<br />Nuclei_Mean_Intensity_mean_RB1_p:  554.42073<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  213.55877<br />Nuclei_Mean_Intensity_mean_RB1_p:   71.30922<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  228.56200<br />Nuclei_Mean_Intensity_mean_RB1_p:  895.34800<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  362.27976<br />Nuclei_Mean_Intensity_mean_RB1_p:  519.12897<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  117.30807<br />Nuclei_Mean_Intensity_mean_RB1_p:  774.45966<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  159.71749<br />Nuclei_Mean_Intensity_mean_RB1_p:  657.94469<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   52.22986<br />Nuclei_Mean_Intensity_mean_RB1_p:  162.60190<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  317.37966<br />Nuclei_Mean_Intensity_mean_RB1_p: 1530.73220<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  116.56380<br />Nuclei_Mean_Intensity_mean_RB1_p:  556.57567<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   73.21849<br />Nuclei_Mean_Intensity_mean_RB1_p:  458.94958<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  313.60559<br />Nuclei_Mean_Intensity_mean_RB1_p: 1543.26353<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  235.41015<br />Nuclei_Mean_Intensity_mean_RB1_p:  806.58774<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  245.33597<br />Nuclei_Mean_Intensity_mean_RB1_p:  561.72742<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  136.98783<br />Nuclei_Mean_Intensity_mean_RB1_p: 1057.69586<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   78.95941<br />Nuclei_Mean_Intensity_mean_RB1_p:  425.01845<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  214.53846<br />Nuclei_Mean_Intensity_mean_RB1_p:  803.72837<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  129.64493<br />Nuclei_Mean_Intensity_mean_RB1_p: 1412.44203<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  117.60417<br />Nuclei_Mean_Intensity_mean_RB1_p: 1425.94583<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  608.04381<br />Nuclei_Mean_Intensity_mean_RB1_p: 1362.77143<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  166.00971<br />Nuclei_Mean_Intensity_mean_RB1_p: 1142.65048<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  175.80435<br />Nuclei_Mean_Intensity_mean_RB1_p:  957.02657<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  289.12375<br />Nuclei_Mean_Intensity_mean_RB1_p:  711.48829<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  105.94709<br />Nuclei_Mean_Intensity_mean_RB1_p:  935.33069<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  115.20442<br />Nuclei_Mean_Intensity_mean_RB1_p:  524.59668<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  151.88080<br />Nuclei_Mean_Intensity_mean_RB1_p:  840.11258<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   90.28903<br />Nuclei_Mean_Intensity_mean_RB1_p:  363.82067<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   91.48700<br />Nuclei_Mean_Intensity_mean_RB1_p:  610.87943<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  194.31874<br />Nuclei_Mean_Intensity_mean_RB1_p:  915.37226<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   74.44204<br />Nuclei_Mean_Intensity_mean_RB1_p:   74.04126<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  110.72995<br />Nuclei_Mean_Intensity_mean_RB1_p:  891.37433<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  110.59191<br />Nuclei_Mean_Intensity_mean_RB1_p:  394.84375<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  110.17935<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.66848<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  180.25145<br />Nuclei_Mean_Intensity_mean_RB1_p:  807.64603<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  298.30724<br />Nuclei_Mean_Intensity_mean_RB1_p:  870.55969<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  107.72123<br />Nuclei_Mean_Intensity_mean_RB1_p:  610.23785<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  359.43939<br />Nuclei_Mean_Intensity_mean_RB1_p:  984.55303<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  317.92016<br />Nuclei_Mean_Intensity_mean_RB1_p: 1141.05389<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  320.27883<br />Nuclei_Mean_Intensity_mean_RB1_p:   33.62479<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  146.73409<br />Nuclei_Mean_Intensity_mean_RB1_p:  515.66721<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  156.15673<br />Nuclei_Mean_Intensity_mean_RB1_p:  832.93598<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   92.16304<br />Nuclei_Mean_Intensity_mean_RB1_p:   22.77446<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  131.22541<br />Nuclei_Mean_Intensity_mean_RB1_p: 2855.89754<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  358.08457<br />Nuclei_Mean_Intensity_mean_RB1_p: 1379.12051<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  139.03831<br />Nuclei_Mean_Intensity_mean_RB1_p:  429.11494<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   62.70300<br />Nuclei_Mean_Intensity_mean_RB1_p:  148.00272<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  235.04618<br />Nuclei_Mean_Intensity_mean_RB1_p:  959.50266<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  160.61485<br />Nuclei_Mean_Intensity_mean_RB1_p:  592.59758<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  185.49454<br />Nuclei_Mean_Intensity_mean_RB1_p:  841.85273<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   95.30787<br />Nuclei_Mean_Intensity_mean_RB1_p:  289.27640<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  155.84805<br />Nuclei_Mean_Intensity_mean_RB1_p:  672.80903<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  252.13762<br />Nuclei_Mean_Intensity_mean_RB1_p:  497.27523<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  580.26177<br />Nuclei_Mean_Intensity_mean_RB1_p:  668.52493<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  139.25150<br />Nuclei_Mean_Intensity_mean_RB1_p:  392.28144<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  134.77320<br />Nuclei_Mean_Intensity_mean_RB1_p:  239.82131<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  248.08044<br />Nuclei_Mean_Intensity_mean_RB1_p:  890.11373<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  341.15630<br />Nuclei_Mean_Intensity_mean_RB1_p:  772.87879<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   81.73558<br />Nuclei_Mean_Intensity_mean_RB1_p:  583.77885<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  119.75316<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.45253<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  171.40422<br />Nuclei_Mean_Intensity_mean_RB1_p:  662.06656<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  398.86293<br />Nuclei_Mean_Intensity_mean_RB1_p:  995.65637<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  384.67683<br />Nuclei_Mean_Intensity_mean_RB1_p:  898.04512<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  244.33162<br />Nuclei_Mean_Intensity_mean_RB1_p:  915.64653<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  101.89896<br />Nuclei_Mean_Intensity_mean_RB1_p:  607.72539<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  216.36765<br />Nuclei_Mean_Intensity_mean_RB1_p:  567.29739<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  127.10145<br />Nuclei_Mean_Intensity_mean_RB1_p:  969.02899<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  168.10232<br />Nuclei_Mean_Intensity_mean_RB1_p:  550.51544<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  110.39429<br />Nuclei_Mean_Intensity_mean_RB1_p:  625.67143<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  238.32676<br />Nuclei_Mean_Intensity_mean_RB1_p:  583.66503<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  115.50330<br />Nuclei_Mean_Intensity_mean_RB1_p:   24.43297<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   80.74825<br />Nuclei_Mean_Intensity_mean_RB1_p:  433.49650<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  261.48220<br />Nuclei_Mean_Intensity_mean_RB1_p:  511.51780<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   92.45324<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.20623<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  202.68659<br />Nuclei_Mean_Intensity_mean_RB1_p:  375.84985<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  539.50000<br />Nuclei_Mean_Intensity_mean_RB1_p:  865.75610<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  132.21714<br />Nuclei_Mean_Intensity_mean_RB1_p:  623.93714<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   99.89145<br />Nuclei_Mean_Intensity_mean_RB1_p:  751.81908<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  232.58756<br />Nuclei_Mean_Intensity_mean_RB1_p:  695.72668<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  230.62000<br />Nuclei_Mean_Intensity_mean_RB1_p:  557.21231<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  158.68153<br />Nuclei_Mean_Intensity_mean_RB1_p:  433.09076<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  134.76056<br />Nuclei_Mean_Intensity_mean_RB1_p:   55.84507<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   90.73301<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.03641<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  190.52863<br />Nuclei_Mean_Intensity_mean_RB1_p:  584.78414<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  175.59839<br />Nuclei_Mean_Intensity_mean_RB1_p:  883.94980<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  151.06888<br />Nuclei_Mean_Intensity_mean_RB1_p:  275.46684<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:   71.49519<br />Nuclei_Mean_Intensity_mean_RB1_p:  704.28205<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  175.67010<br />Nuclei_Mean_Intensity_mean_RB1_p:  425.50000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  213.54399<br />Nuclei_Mean_Intensity_mean_RB1_p:  622.77859<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  108.47956<br />Nuclei_Mean_Intensity_mean_RB1_p:  665.51771<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  100.48297<br />Nuclei_Mean_Intensity_mean_RB1_p:  899.45511<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  116.85421<br />Nuclei_Mean_Intensity_mean_RB1_p:  575.82460<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Nuclei_Mean_Intensity_mean_CCNA2:  192.47101<br />Nuclei_Mean_Intensity_mean_RB1_p:  771.51159<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  243.96825<br />Nuclei_Mean_Intensity_mean_RB1_p:  529.05952<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  341.16605<br />Nuclei_Mean_Intensity_mean_RB1_p:  407.95018<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  300.32552<br />Nuclei_Mean_Intensity_mean_RB1_p:  970.11035<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  205.33187<br />Nuclei_Mean_Intensity_mean_RB1_p:  806.89035<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  300.97265<br />Nuclei_Mean_Intensity_mean_RB1_p: 1003.81538<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  361.89153<br />Nuclei_Mean_Intensity_mean_RB1_p:  559.42989<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   94.54490<br />Nuclei_Mean_Intensity_mean_RB1_p:  251.53878<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  207.15489<br />Nuclei_Mean_Intensity_mean_RB1_p:  599.40451<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  241.70434<br />Nuclei_Mean_Intensity_mean_RB1_p:  709.76216<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  149.46122<br />Nuclei_Mean_Intensity_mean_RB1_p:  368.46531<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  276.72680<br />Nuclei_Mean_Intensity_mean_RB1_p:  763.42440<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  110.55599<br />Nuclei_Mean_Intensity_mean_RB1_p:  275.91552<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  384.19817<br />Nuclei_Mean_Intensity_mean_RB1_p: 1081.71037<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  362.44071<br />Nuclei_Mean_Intensity_mean_RB1_p:  534.48319<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  160.29027<br />Nuclei_Mean_Intensity_mean_RB1_p:  654.32695<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   80.25513<br />Nuclei_Mean_Intensity_mean_RB1_p:  626.44575<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  101.02357<br />Nuclei_Mean_Intensity_mean_RB1_p:  367.81482<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  591.37381<br />Nuclei_Mean_Intensity_mean_RB1_p:  711.08729<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   72.22253<br />Nuclei_Mean_Intensity_mean_RB1_p:  174.62912<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  174.38789<br />Nuclei_Mean_Intensity_mean_RB1_p:  446.07856<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  149.90290<br />Nuclei_Mean_Intensity_mean_RB1_p:   29.66078<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  119.71729<br />Nuclei_Mean_Intensity_mean_RB1_p:  409.86682<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  274.07081<br />Nuclei_Mean_Intensity_mean_RB1_p:  635.28179<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  140.93582<br />Nuclei_Mean_Intensity_mean_RB1_p:  493.93996<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  159.87052<br />Nuclei_Mean_Intensity_mean_RB1_p:  436.94422<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  123.51569<br />Nuclei_Mean_Intensity_mean_RB1_p:  616.39910<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  238.02632<br />Nuclei_Mean_Intensity_mean_RB1_p:  677.88346<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  102.53333<br />Nuclei_Mean_Intensity_mean_RB1_p:  588.04630<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  150.21986<br />Nuclei_Mean_Intensity_mean_RB1_p:  983.26773<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  126.04910<br />Nuclei_Mean_Intensity_mean_RB1_p:  964.57881<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  112.62579<br />Nuclei_Mean_Intensity_mean_RB1_p:  848.11950<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   96.07676<br />Nuclei_Mean_Intensity_mean_RB1_p:  962.78891<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   88.06284<br />Nuclei_Mean_Intensity_mean_RB1_p: 1303.01913<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:   93.15427<br />Nuclei_Mean_Intensity_mean_RB1_p: 1122.13223<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  194.79667<br />Nuclei_Mean_Intensity_mean_RB1_p: 1064.31238<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  360.02215<br />Nuclei_Mean_Intensity_mean_RB1_p: 1034.18058<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  365.12448<br />Nuclei_Mean_Intensity_mean_RB1_p:  182.23790<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  144.55206<br />Nuclei_Mean_Intensity_mean_RB1_p:  880.75787<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  317.34228<br />Nuclei_Mean_Intensity_mean_RB1_p:  944.78691<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  140.99278<br />Nuclei_Mean_Intensity_mean_RB1_p:  462.35379<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  122.29167<br />Nuclei_Mean_Intensity_mean_RB1_p:  904.80208<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  202.98679<br />Nuclei_Mean_Intensity_mean_RB1_p:  633.58113<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  204.76060<br />Nuclei_Mean_Intensity_mean_RB1_p: 1157.66833<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  119.53333<br />Nuclei_Mean_Intensity_mean_RB1_p:  402.73750<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  194.01339<br />Nuclei_Mean_Intensity_mean_RB1_p:  936.52679<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  167.90141<br />Nuclei_Mean_Intensity_mean_RB1_p:  584.16620<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  127.26686<br />Nuclei_Mean_Intensity_mean_RB1_p:   52.01760<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  101.27624<br />Nuclei_Mean_Intensity_mean_RB1_p:  341.71547<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  140.54645<br />Nuclei_Mean_Intensity_mean_RB1_p: 1581.83880<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  206.70667<br />Nuclei_Mean_Intensity_mean_RB1_p:  548.60800<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  220.56650<br />Nuclei_Mean_Intensity_mean_RB1_p: 1852.19212<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  119.66452<br />Nuclei_Mean_Intensity_mean_RB1_p:  615.31828<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  156.22699<br />Nuclei_Mean_Intensity_mean_RB1_p:  707.65644<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  159.90856<br />Nuclei_Mean_Intensity_mean_RB1_p:  163.92625<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  153.17688<br />Nuclei_Mean_Intensity_mean_RB1_p:  460.47772<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  136.93243<br />Nuclei_Mean_Intensity_mean_RB1_p:  563.38224<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  164.17424<br />Nuclei_Mean_Intensity_mean_RB1_p: 1104.63384<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  148.96707<br />Nuclei_Mean_Intensity_mean_RB1_p:  891.41075<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  158.12424<br />Nuclei_Mean_Intensity_mean_RB1_p: 1237.91039<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  168.37402<br />Nuclei_Mean_Intensity_mean_RB1_p:  565.83099<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  156.46787<br />Nuclei_Mean_Intensity_mean_RB1_p:  776.22691<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  171.94102<br />Nuclei_Mean_Intensity_mean_RB1_p:  953.35121<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  122.48775<br />Nuclei_Mean_Intensity_mean_RB1_p:  463.55011<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  149.88146<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.86626<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  118.01653<br />Nuclei_Mean_Intensity_mean_RB1_p:  780.33678<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  140.08559<br />Nuclei_Mean_Intensity_mean_RB1_p:  390.97523<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  107.61219<br />Nuclei_Mean_Intensity_mean_RB1_p:  318.19668<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  151.36919<br />Nuclei_Mean_Intensity_mean_RB1_p:   26.46455<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  126.74555<br />Nuclei_Mean_Intensity_mean_RB1_p:  419.56234<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  159.77941<br />Nuclei_Mean_Intensity_mean_RB1_p:  916.65686<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2: 1367.65275<br />Nuclei_Mean_Intensity_mean_RB1_p: 1073.82543<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  122.31085<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.74780<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  131.54415<br />Nuclei_Mean_Intensity_mean_RB1_p:  857.42482<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  582.86511<br />Nuclei_Mean_Intensity_mean_RB1_p:  533.32914<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  232.69691<br />Nuclei_Mean_Intensity_mean_RB1_p:  540.17182<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  157.14213<br />Nuclei_Mean_Intensity_mean_RB1_p:  714.89594<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  175.36111<br />Nuclei_Mean_Intensity_mean_RB1_p:  485.26157<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  251.32945<br />Nuclei_Mean_Intensity_mean_RB1_p:  653.08985<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  148.57245<br />Nuclei_Mean_Intensity_mean_RB1_p:  674.03563<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  378.07170<br />Nuclei_Mean_Intensity_mean_RB1_p:  668.57170<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  118.14421<br />Nuclei_Mean_Intensity_mean_RB1_p:  402.88416<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  970.59392<br />Nuclei_Mean_Intensity_mean_RB1_p: 1959.45080<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2: 1334.84772<br />Nuclei_Mean_Intensity_mean_RB1_p:  585.73773<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  155.20896<br />Nuclei_Mean_Intensity_mean_RB1_p:  583.05638<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  216.64875<br />Nuclei_Mean_Intensity_mean_RB1_p:  617.29928<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  330.50000<br />Nuclei_Mean_Intensity_mean_RB1_p:  640.24684<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  178.82609<br />Nuclei_Mean_Intensity_mean_RB1_p:  832.66567<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  149.08402<br />Nuclei_Mean_Intensity_mean_RB1_p:  372.73806<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  140.20496<br />Nuclei_Mean_Intensity_mean_RB1_p:  372.68016<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  138.46269<br />Nuclei_Mean_Intensity_mean_RB1_p:  787.07214<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  101.19375<br />Nuclei_Mean_Intensity_mean_RB1_p:   52.13125<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  143.16573<br />Nuclei_Mean_Intensity_mean_RB1_p:   36.33146<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  141.62000<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.31750<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  132.81789<br />Nuclei_Mean_Intensity_mean_RB1_p:   29.08943<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  159.30415<br />Nuclei_Mean_Intensity_mean_RB1_p:  354.91705<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  125.86190<br />Nuclei_Mean_Intensity_mean_RB1_p:  988.14310<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  142.57330<br />Nuclei_Mean_Intensity_mean_RB1_p:  578.66958<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  165.74651<br />Nuclei_Mean_Intensity_mean_RB1_p:   22.81395<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  187.33586<br />Nuclei_Mean_Intensity_mean_RB1_p:  905.78063<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  141.24935<br />Nuclei_Mean_Intensity_mean_RB1_p:  655.78182<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  245.71664<br />Nuclei_Mean_Intensity_mean_RB1_p:  341.85832<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  135.33894<br />Nuclei_Mean_Intensity_mean_RB1_p:  535.94398<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  164.10025<br />Nuclei_Mean_Intensity_mean_RB1_p:  314.48872<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  623.46857<br />Nuclei_Mean_Intensity_mean_RB1_p:  574.64190<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  112.82558<br />Nuclei_Mean_Intensity_mean_RB1_p:  484.06202<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  138.28527<br />Nuclei_Mean_Intensity_mean_RB1_p:  116.05329<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  239.91350<br />Nuclei_Mean_Intensity_mean_RB1_p:  922.40830<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  184.37000<br />Nuclei_Mean_Intensity_mean_RB1_p:  748.22333<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  398.64780<br />Nuclei_Mean_Intensity_mean_RB1_p:  718.29979<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  116.15599<br />Nuclei_Mean_Intensity_mean_RB1_p:   62.93593<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  171.00613<br />Nuclei_Mean_Intensity_mean_RB1_p:  428.14701<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  195.48231<br />Nuclei_Mean_Intensity_mean_RB1_p: 1093.88082<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  384.54950<br />Nuclei_Mean_Intensity_mean_RB1_p:  384.60832<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  225.97026<br />Nuclei_Mean_Intensity_mean_RB1_p:  436.80111<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  169.07053<br />Nuclei_Mean_Intensity_mean_RB1_p:  528.70219<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  146.13760<br />Nuclei_Mean_Intensity_mean_RB1_p:   35.59690<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  230.39012<br />Nuclei_Mean_Intensity_mean_RB1_p:  510.71721<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  138.18525<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.17626<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  337.91047<br />Nuclei_Mean_Intensity_mean_RB1_p: 1043.44766<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  177.44828<br />Nuclei_Mean_Intensity_mean_RB1_p:  839.09770<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  211.87631<br />Nuclei_Mean_Intensity_mean_RB1_p: 1053.24216<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  121.21729<br />Nuclei_Mean_Intensity_mean_RB1_p:  766.20561<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  211.24242<br />Nuclei_Mean_Intensity_mean_RB1_p:  998.05742<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Nuclei_Mean_Intensity_mean_CCNA2:  141.40333<br />Nuclei_Mean_Intensity_mean_RB1_p: 1032.33472<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  102.53133<br />Nuclei_Mean_Intensity_mean_RB1_p:  216.39599<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  112.79043<br />Nuclei_Mean_Intensity_mean_RB1_p:  798.15718<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  143.06769<br />Nuclei_Mean_Intensity_mean_RB1_p:   24.25546<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  168.81667<br />Nuclei_Mean_Intensity_mean_RB1_p:  781.27222<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   87.75296<br />Nuclei_Mean_Intensity_mean_RB1_p:  395.19170<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  232.47432<br />Nuclei_Mean_Intensity_mean_RB1_p:  401.71148<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  225.82297<br />Nuclei_Mean_Intensity_mean_RB1_p:  668.92983<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  138.43237<br />Nuclei_Mean_Intensity_mean_RB1_p:  582.59420<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  114.11670<br />Nuclei_Mean_Intensity_mean_RB1_p:  424.70709<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  119.04701<br />Nuclei_Mean_Intensity_mean_RB1_p:  374.07906<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  167.91774<br />Nuclei_Mean_Intensity_mean_RB1_p: 1153.50900<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  271.85172<br />Nuclei_Mean_Intensity_mean_RB1_p:  732.83725<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  124.22207<br />Nuclei_Mean_Intensity_mean_RB1_p:   35.15363<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   91.77641<br />Nuclei_Mean_Intensity_mean_RB1_p:  504.86978<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  310.79249<br />Nuclei_Mean_Intensity_mean_RB1_p:  708.11807<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   94.56749<br />Nuclei_Mean_Intensity_mean_RB1_p:   26.83471<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  451.61411<br />Nuclei_Mean_Intensity_mean_RB1_p:  723.03319<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  225.67269<br />Nuclei_Mean_Intensity_mean_RB1_p:  726.77758<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  306.14349<br />Nuclei_Mean_Intensity_mean_RB1_p: 1418.61810<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   90.62500<br />Nuclei_Mean_Intensity_mean_RB1_p:  439.98558<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  454.22958<br />Nuclei_Mean_Intensity_mean_RB1_p:  441.09934<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   71.43348<br />Nuclei_Mean_Intensity_mean_RB1_p:  597.74464<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  182.24059<br />Nuclei_Mean_Intensity_mean_RB1_p:   32.13808<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  265.13417<br />Nuclei_Mean_Intensity_mean_RB1_p: 1405.41682<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  105.90751<br />Nuclei_Mean_Intensity_mean_RB1_p:  559.29480<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  113.48068<br />Nuclei_Mean_Intensity_mean_RB1_p:  294.24396<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  178.26654<br />Nuclei_Mean_Intensity_mean_RB1_p:  810.76265<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  227.66038<br />Nuclei_Mean_Intensity_mean_RB1_p:  326.63207<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  136.60000<br />Nuclei_Mean_Intensity_mean_RB1_p:  562.14706<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   92.76966<br />Nuclei_Mean_Intensity_mean_RB1_p:  483.38389<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  133.14349<br />Nuclei_Mean_Intensity_mean_RB1_p:  461.98455<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  122.59170<br />Nuclei_Mean_Intensity_mean_RB1_p:  564.28603<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  224.09405<br />Nuclei_Mean_Intensity_mean_RB1_p:  571.80614<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  252.24536<br />Nuclei_Mean_Intensity_mean_RB1_p:  906.60913<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  111.08415<br />Nuclei_Mean_Intensity_mean_RB1_p:  517.38356<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  106.50131<br />Nuclei_Mean_Intensity_mean_RB1_p:  439.71802<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   92.91750<br />Nuclei_Mean_Intensity_mean_RB1_p:  419.87500<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  135.20805<br />Nuclei_Mean_Intensity_mean_RB1_p:  917.28188<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  196.01173<br />Nuclei_Mean_Intensity_mean_RB1_p:  691.14076<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  127.17344<br />Nuclei_Mean_Intensity_mean_RB1_p: 1010.51490<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  178.43602<br />Nuclei_Mean_Intensity_mean_RB1_p:  494.44313<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  115.35098<br />Nuclei_Mean_Intensity_mean_RB1_p:  356.94150<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  184.27342<br />Nuclei_Mean_Intensity_mean_RB1_p:  394.49367<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  389.81538<br />Nuclei_Mean_Intensity_mean_RB1_p:  635.96346<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  154.00748<br />Nuclei_Mean_Intensity_mean_RB1_p: 1144.48878<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   97.52210<br />Nuclei_Mean_Intensity_mean_RB1_p:  442.24842<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  164.17462<br />Nuclei_Mean_Intensity_mean_RB1_p:  254.21692<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  171.07991<br />Nuclei_Mean_Intensity_mean_RB1_p:   23.97840<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  202.11515<br />Nuclei_Mean_Intensity_mean_RB1_p:  628.44485<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  124.93435<br />Nuclei_Mean_Intensity_mean_RB1_p:  557.86214<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  260.85260<br />Nuclei_Mean_Intensity_mean_RB1_p:  689.19096<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  154.73771<br />Nuclei_Mean_Intensity_mean_RB1_p:  521.08431<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  122.32998<br />Nuclei_Mean_Intensity_mean_RB1_p:  479.75567<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   46.80797<br />Nuclei_Mean_Intensity_mean_RB1_p:  485.17029<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  120.05587<br />Nuclei_Mean_Intensity_mean_RB1_p:  520.32402<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  174.98280<br />Nuclei_Mean_Intensity_mean_RB1_p:  232.90418<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  195.37259<br />Nuclei_Mean_Intensity_mean_RB1_p:  500.52896<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   94.98155<br />Nuclei_Mean_Intensity_mean_RB1_p:  841.69742<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   85.45087<br />Nuclei_Mean_Intensity_mean_RB1_p:  587.22832<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  212.39900<br />Nuclei_Mean_Intensity_mean_RB1_p:   29.86035<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  177.99542<br />Nuclei_Mean_Intensity_mean_RB1_p: 1087.96567<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  203.73058<br />Nuclei_Mean_Intensity_mean_RB1_p:  849.36364<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  138.61872<br />Nuclei_Mean_Intensity_mean_RB1_p:  633.65753<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  141.66049<br />Nuclei_Mean_Intensity_mean_RB1_p:  133.51235<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  133.84701<br />Nuclei_Mean_Intensity_mean_RB1_p:  646.27938<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  108.38397<br />Nuclei_Mean_Intensity_mean_RB1_p: 1149.96625<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  143.34291<br />Nuclei_Mean_Intensity_mean_RB1_p:  758.19157<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  252.52539<br />Nuclei_Mean_Intensity_mean_RB1_p:  945.61821<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  133.02972<br />Nuclei_Mean_Intensity_mean_RB1_p:  587.72808<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  100.44983<br />Nuclei_Mean_Intensity_mean_RB1_p:  551.07266<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  207.26255<br />Nuclei_Mean_Intensity_mean_RB1_p:  595.67182<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  135.81182<br />Nuclei_Mean_Intensity_mean_RB1_p:  743.78775<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  189.78027<br />Nuclei_Mean_Intensity_mean_RB1_p:  712.56727<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  224.99408<br />Nuclei_Mean_Intensity_mean_RB1_p:  852.07889<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  569.42785<br />Nuclei_Mean_Intensity_mean_RB1_p: 1719.62248<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  156.30740<br />Nuclei_Mean_Intensity_mean_RB1_p:  620.37192<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  187.22026<br />Nuclei_Mean_Intensity_mean_RB1_p: 1101.56828<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  258.09797<br />Nuclei_Mean_Intensity_mean_RB1_p:  566.75169<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   93.17429<br />Nuclei_Mean_Intensity_mean_RB1_p:  392.68857<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  141.64389<br />Nuclei_Mean_Intensity_mean_RB1_p:  447.96066<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   94.29383<br />Nuclei_Mean_Intensity_mean_RB1_p:  215.53333<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   87.52404<br />Nuclei_Mean_Intensity_mean_RB1_p:  700.65385<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   89.76768<br />Nuclei_Mean_Intensity_mean_RB1_p:  464.85606<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  107.31544<br />Nuclei_Mean_Intensity_mean_RB1_p: 1032.32662<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  205.17857<br />Nuclei_Mean_Intensity_mean_RB1_p:  537.11526<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  361.31119<br />Nuclei_Mean_Intensity_mean_RB1_p: 1684.09867<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  150.01663<br />Nuclei_Mean_Intensity_mean_RB1_p:  422.48441<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  199.91385<br />Nuclei_Mean_Intensity_mean_RB1_p:  781.36993<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  259.73101<br />Nuclei_Mean_Intensity_mean_RB1_p:  852.25257<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  146.70157<br />Nuclei_Mean_Intensity_mean_RB1_p:  619.06283<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  701.39322<br />Nuclei_Mean_Intensity_mean_RB1_p:  414.95763<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  106.50128<br />Nuclei_Mean_Intensity_mean_RB1_p:   36.31620<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  133.43492<br />Nuclei_Mean_Intensity_mean_RB1_p:   54.93968<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  237.88732<br />Nuclei_Mean_Intensity_mean_RB1_p:  761.11424<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  485.65168<br />Nuclei_Mean_Intensity_mean_RB1_p:  997.27715<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   98.30220<br />Nuclei_Mean_Intensity_mean_RB1_p:  167.36264<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   98.76359<br />Nuclei_Mean_Intensity_mean_RB1_p:  209.61466<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   91.64000<br />Nuclei_Mean_Intensity_mean_RB1_p:  525.91467<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   98.52830<br />Nuclei_Mean_Intensity_mean_RB1_p:  471.14151<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  132.28350<br />Nuclei_Mean_Intensity_mean_RB1_p:  588.60825<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  656.99849<br />Nuclei_Mean_Intensity_mean_RB1_p:  924.29669<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   79.15751<br />Nuclei_Mean_Intensity_mean_RB1_p:  715.91941<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  150.74148<br />Nuclei_Mean_Intensity_mean_RB1_p:  443.67335<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:   70.44528<br />Nuclei_Mean_Intensity_mean_RB1_p:  715.30566<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  286.48118<br />Nuclei_Mean_Intensity_mean_RB1_p:  487.95968<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  119.42253<br />Nuclei_Mean_Intensity_mean_RB1_p:  584.65070<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Nuclei_Mean_Intensity_mean_CCNA2:  114.93548<br />Nuclei_Mean_Intensity_mean_RB1_p:  566.94541<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  161.30508<br />Nuclei_Mean_Intensity_mean_RB1_p:  832.38771<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  157.51045<br />Nuclei_Mean_Intensity_mean_RB1_p:  589.77612<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  203.16588<br />Nuclei_Mean_Intensity_mean_RB1_p:  734.26761<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  176.40049<br />Nuclei_Mean_Intensity_mean_RB1_p:  654.44175<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  164.51659<br />Nuclei_Mean_Intensity_mean_RB1_p:  438.44313<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  132.66237<br />Nuclei_Mean_Intensity_mean_RB1_p:  629.75258<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  106.96375<br />Nuclei_Mean_Intensity_mean_RB1_p:  415.29607<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  249.90426<br />Nuclei_Mean_Intensity_mean_RB1_p:  549.32624<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  174.19744<br />Nuclei_Mean_Intensity_mean_RB1_p: 1046.21026<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  186.33702<br />Nuclei_Mean_Intensity_mean_RB1_p:  902.76519<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  105.41515<br />Nuclei_Mean_Intensity_mean_RB1_p:  674.58030<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  162.29002<br />Nuclei_Mean_Intensity_mean_RB1_p:  760.43852<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  241.31152<br />Nuclei_Mean_Intensity_mean_RB1_p:  442.91230<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  117.80000<br />Nuclei_Mean_Intensity_mean_RB1_p:  665.84884<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  191.61824<br />Nuclei_Mean_Intensity_mean_RB1_p:  860.65878<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  173.94210<br />Nuclei_Mean_Intensity_mean_RB1_p:  664.06579<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  122.46633<br />Nuclei_Mean_Intensity_mean_RB1_p:  470.34414<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  217.21074<br />Nuclei_Mean_Intensity_mean_RB1_p:   29.16942<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  129.96831<br />Nuclei_Mean_Intensity_mean_RB1_p:  657.20422<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  262.99751<br />Nuclei_Mean_Intensity_mean_RB1_p:  694.08479<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  313.82936<br />Nuclei_Mean_Intensity_mean_RB1_p:  566.46239<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  157.13189<br />Nuclei_Mean_Intensity_mean_RB1_p:  675.15354<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  424.11371<br />Nuclei_Mean_Intensity_mean_RB1_p:  578.59030<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  172.48846<br />Nuclei_Mean_Intensity_mean_RB1_p:   26.68654<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  151.44444<br />Nuclei_Mean_Intensity_mean_RB1_p:  694.06746<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   85.24315<br />Nuclei_Mean_Intensity_mean_RB1_p:  370.99315<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  155.65101<br />Nuclei_Mean_Intensity_mean_RB1_p:  268.30872<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  146.25223<br />Nuclei_Mean_Intensity_mean_RB1_p:  484.85163<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   83.83246<br />Nuclei_Mean_Intensity_mean_RB1_p:  306.30890<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  105.89214<br />Nuclei_Mean_Intensity_mean_RB1_p:  492.89945<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  120.35049<br />Nuclei_Mean_Intensity_mean_RB1_p:  610.22549<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   96.31393<br />Nuclei_Mean_Intensity_mean_RB1_p:  313.84823<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  230.77993<br />Nuclei_Mean_Intensity_mean_RB1_p:  559.83451<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  182.68848<br />Nuclei_Mean_Intensity_mean_RB1_p: 1042.59948<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  108.20981<br />Nuclei_Mean_Intensity_mean_RB1_p:   43.83106<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  110.11429<br />Nuclei_Mean_Intensity_mean_RB1_p:  589.22857<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   64.70742<br />Nuclei_Mean_Intensity_mean_RB1_p:   40.26856<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  133.30361<br />Nuclei_Mean_Intensity_mean_RB1_p:  256.34535<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  325.31541<br />Nuclei_Mean_Intensity_mean_RB1_p:  380.58781<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  623.15070<br />Nuclei_Mean_Intensity_mean_RB1_p: 1296.01549<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  133.03636<br />Nuclei_Mean_Intensity_mean_RB1_p: 1925.71169<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  338.76099<br />Nuclei_Mean_Intensity_mean_RB1_p:  761.33843<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  147.17967<br />Nuclei_Mean_Intensity_mean_RB1_p:  776.94090<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  157.60656<br />Nuclei_Mean_Intensity_mean_RB1_p:  115.84777<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  115.27901<br />Nuclei_Mean_Intensity_mean_RB1_p:  440.57182<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  149.80734<br />Nuclei_Mean_Intensity_mean_RB1_p:  960.60092<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  196.31467<br />Nuclei_Mean_Intensity_mean_RB1_p:  324.63733<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  215.08918<br />Nuclei_Mean_Intensity_mean_RB1_p: 1008.07970<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  126.61845<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.51122<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  157.54054<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.26781<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  146.96838<br />Nuclei_Mean_Intensity_mean_RB1_p:  466.98024<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  228.14691<br />Nuclei_Mean_Intensity_mean_RB1_p:  570.09021<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  724.67407<br />Nuclei_Mean_Intensity_mean_RB1_p: 1054.23259<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  152.17661<br />Nuclei_Mean_Intensity_mean_RB1_p:  664.72554<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  238.65962<br />Nuclei_Mean_Intensity_mean_RB1_p:  821.42723<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  225.72666<br />Nuclei_Mean_Intensity_mean_RB1_p:  794.38836<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  156.98929<br />Nuclei_Mean_Intensity_mean_RB1_p:  859.20771<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   95.45937<br />Nuclei_Mean_Intensity_mean_RB1_p:  750.72187<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  112.66458<br />Nuclei_Mean_Intensity_mean_RB1_p:  677.23198<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  283.52589<br />Nuclei_Mean_Intensity_mean_RB1_p:  729.42557<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  257.92161<br />Nuclei_Mean_Intensity_mean_RB1_p:  852.01483<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  178.98254<br />Nuclei_Mean_Intensity_mean_RB1_p:  683.86783<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  129.40151<br />Nuclei_Mean_Intensity_mean_RB1_p:  617.16667<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  118.07711<br />Nuclei_Mean_Intensity_mean_RB1_p:  438.43035<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  137.37461<br />Nuclei_Mean_Intensity_mean_RB1_p: 1052.94118<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  129.80044<br />Nuclei_Mean_Intensity_mean_RB1_p:  391.98026<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  170.78876<br />Nuclei_Mean_Intensity_mean_RB1_p:  895.18652<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  161.00862<br />Nuclei_Mean_Intensity_mean_RB1_p:  806.90517<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  135.98370<br />Nuclei_Mean_Intensity_mean_RB1_p:  593.62228<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  142.29648<br />Nuclei_Mean_Intensity_mean_RB1_p: 1024.90452<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  102.06333<br />Nuclei_Mean_Intensity_mean_RB1_p:  403.03333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  131.13525<br />Nuclei_Mean_Intensity_mean_RB1_p:  544.94672<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  181.76104<br />Nuclei_Mean_Intensity_mean_RB1_p:  415.72289<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  105.94458<br />Nuclei_Mean_Intensity_mean_RB1_p:  928.51807<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  143.35813<br />Nuclei_Mean_Intensity_mean_RB1_p:  744.44353<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  113.98855<br />Nuclei_Mean_Intensity_mean_RB1_p:  138.60496<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  198.69595<br />Nuclei_Mean_Intensity_mean_RB1_p:  654.04054<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  102.00604<br />Nuclei_Mean_Intensity_mean_RB1_p:  612.52266<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  364.71603<br />Nuclei_Mean_Intensity_mean_RB1_p:  491.51450<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  113.91563<br />Nuclei_Mean_Intensity_mean_RB1_p:  517.96563<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  128.33241<br />Nuclei_Mean_Intensity_mean_RB1_p:  534.40166<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   89.45299<br />Nuclei_Mean_Intensity_mean_RB1_p:  494.03205<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  233.25093<br />Nuclei_Mean_Intensity_mean_RB1_p:  641.10966<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  252.73282<br />Nuclei_Mean_Intensity_mean_RB1_p: 1109.22519<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  125.04604<br />Nuclei_Mean_Intensity_mean_RB1_p:  887.69565<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  198.04048<br />Nuclei_Mean_Intensity_mean_RB1_p:  482.63718<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  272.75198<br />Nuclei_Mean_Intensity_mean_RB1_p:  748.80569<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  156.03618<br />Nuclei_Mean_Intensity_mean_RB1_p:  321.57364<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  384.46097<br />Nuclei_Mean_Intensity_mean_RB1_p: 1375.67100<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  439.00997<br />Nuclei_Mean_Intensity_mean_RB1_p: 1866.82890<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  108.61404<br />Nuclei_Mean_Intensity_mean_RB1_p:  420.59503<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  157.07392<br />Nuclei_Mean_Intensity_mean_RB1_p:  431.51745<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  247.01290<br />Nuclei_Mean_Intensity_mean_RB1_p:  250.31183<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  114.70284<br />Nuclei_Mean_Intensity_mean_RB1_p:  654.63566<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  404.60813<br />Nuclei_Mean_Intensity_mean_RB1_p:  648.68795<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  132.69377<br />Nuclei_Mean_Intensity_mean_RB1_p:  655.33604<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  154.54267<br />Nuclei_Mean_Intensity_mean_RB1_p:  626.01094<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  161.17077<br />Nuclei_Mean_Intensity_mean_RB1_p:   25.07394<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  257.98488<br />Nuclei_Mean_Intensity_mean_RB1_p:  882.52174<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  253.21875<br />Nuclei_Mean_Intensity_mean_RB1_p:  920.52303<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  268.58491<br />Nuclei_Mean_Intensity_mean_RB1_p: 1028.71132<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  188.23266<br />Nuclei_Mean_Intensity_mean_RB1_p: 1081.02023<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   88.17414<br />Nuclei_Mean_Intensity_mean_RB1_p:  682.46702<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  256.20823<br />Nuclei_Mean_Intensity_mean_RB1_p:  404.52785<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  552.05893<br />Nuclei_Mean_Intensity_mean_RB1_p:  554.93554<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  194.65571<br />Nuclei_Mean_Intensity_mean_RB1_p:  434.46035<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  292.40909<br />Nuclei_Mean_Intensity_mean_RB1_p:  536.81993<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  251.79328<br />Nuclei_Mean_Intensity_mean_RB1_p: 1686.03193<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  119.76045<br />Nuclei_Mean_Intensity_mean_RB1_p:  706.82451<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  129.50943<br />Nuclei_Mean_Intensity_mean_RB1_p:   34.48113<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  158.53073<br />Nuclei_Mean_Intensity_mean_RB1_p:  466.01199<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  529.65524<br />Nuclei_Mean_Intensity_mean_RB1_p:  731.05040<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  434.16761<br />Nuclei_Mean_Intensity_mean_RB1_p:  519.28028<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  133.04279<br />Nuclei_Mean_Intensity_mean_RB1_p:  389.16441<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  123.36923<br />Nuclei_Mean_Intensity_mean_RB1_p: 1310.05128<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  285.65271<br />Nuclei_Mean_Intensity_mean_RB1_p:  927.49913<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  155.18564<br />Nuclei_Mean_Intensity_mean_RB1_p:  641.89317<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  237.46725<br />Nuclei_Mean_Intensity_mean_RB1_p:  958.47817<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  240.14395<br />Nuclei_Mean_Intensity_mean_RB1_p:  890.14012<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   96.44730<br />Nuclei_Mean_Intensity_mean_RB1_p:  268.28278<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  899.73835<br />Nuclei_Mean_Intensity_mean_RB1_p: 1340.20451<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  142.89000<br />Nuclei_Mean_Intensity_mean_RB1_p:  934.19667<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  152.56338<br />Nuclei_Mean_Intensity_mean_RB1_p:  623.78404<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  174.36566<br />Nuclei_Mean_Intensity_mean_RB1_p:  551.19192<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  148.19636<br />Nuclei_Mean_Intensity_mean_RB1_p: 1760.97455<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  213.75942<br />Nuclei_Mean_Intensity_mean_RB1_p:  931.99641<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   91.51531<br />Nuclei_Mean_Intensity_mean_RB1_p:  380.01020<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  136.15521<br />Nuclei_Mean_Intensity_mean_RB1_p:  910.63636<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  184.65177<br />Nuclei_Mean_Intensity_mean_RB1_p:  781.21177<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  189.55165<br />Nuclei_Mean_Intensity_mean_RB1_p:  499.71694<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:  136.86100<br />Nuclei_Mean_Intensity_mean_RB1_p: 1028.44208<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Nuclei_Mean_Intensity_mean_CCNA2:   92.13312<br />Nuclei_Mean_Intensity_mean_RB1_p:  851.53571<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  209.33394<br />Nuclei_Mean_Intensity_mean_RB1_p:  517.03085<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  204.31482<br />Nuclei_Mean_Intensity_mean_RB1_p:  835.86027<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  158.87631<br />Nuclei_Mean_Intensity_mean_RB1_p: 1108.63941<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  165.47156<br />Nuclei_Mean_Intensity_mean_RB1_p:  701.59953<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  517.70382<br />Nuclei_Mean_Intensity_mean_RB1_p:  968.26412<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   95.91417<br />Nuclei_Mean_Intensity_mean_RB1_p:  715.52695<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  101.80083<br />Nuclei_Mean_Intensity_mean_RB1_p:  590.15076<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  114.46402<br />Nuclei_Mean_Intensity_mean_RB1_p:  485.41935<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  114.17347<br />Nuclei_Mean_Intensity_mean_RB1_p:  634.40561<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  223.24510<br />Nuclei_Mean_Intensity_mean_RB1_p:  924.85948<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   92.64619<br />Nuclei_Mean_Intensity_mean_RB1_p:  647.82555<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   86.96166<br />Nuclei_Mean_Intensity_mean_RB1_p:  399.70927<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  156.53775<br />Nuclei_Mean_Intensity_mean_RB1_p:  428.43914<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   95.15449<br />Nuclei_Mean_Intensity_mean_RB1_p:  683.87474<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  115.66391<br />Nuclei_Mean_Intensity_mean_RB1_p:  342.47658<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  138.93474<br />Nuclei_Mean_Intensity_mean_RB1_p:  456.82918<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  112.03327<br />Nuclei_Mean_Intensity_mean_RB1_p:  560.19178<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  109.46851<br />Nuclei_Mean_Intensity_mean_RB1_p:  649.38035<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  281.72690<br />Nuclei_Mean_Intensity_mean_RB1_p: 1274.39836<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  190.07797<br />Nuclei_Mean_Intensity_mean_RB1_p:  356.46979<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  123.17333<br />Nuclei_Mean_Intensity_mean_RB1_p:  271.70444<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   86.16897<br />Nuclei_Mean_Intensity_mean_RB1_p:  373.36207<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  272.48953<br />Nuclei_Mean_Intensity_mean_RB1_p:  594.22705<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  131.83536<br />Nuclei_Mean_Intensity_mean_RB1_p:  700.60189<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   63.88000<br />Nuclei_Mean_Intensity_mean_RB1_p:  734.73455<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  180.62104<br />Nuclei_Mean_Intensity_mean_RB1_p:  497.27883<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  124.84637<br />Nuclei_Mean_Intensity_mean_RB1_p:  727.45531<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  113.34875<br />Nuclei_Mean_Intensity_mean_RB1_p:  165.03915<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  147.91513<br />Nuclei_Mean_Intensity_mean_RB1_p:  763.73247<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  133.64062<br />Nuclei_Mean_Intensity_mean_RB1_p:  586.17708<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  171.80469<br />Nuclei_Mean_Intensity_mean_RB1_p:  527.89648<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  133.06295<br />Nuclei_Mean_Intensity_mean_RB1_p: 1091.58353<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  167.33333<br />Nuclei_Mean_Intensity_mean_RB1_p:  395.51685<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  268.69300<br />Nuclei_Mean_Intensity_mean_RB1_p:  844.78104<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  239.13213<br />Nuclei_Mean_Intensity_mean_RB1_p:  771.20270<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  163.33992<br />Nuclei_Mean_Intensity_mean_RB1_p:  998.03821<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  116.03333<br />Nuclei_Mean_Intensity_mean_RB1_p:  814.15476<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  166.91980<br />Nuclei_Mean_Intensity_mean_RB1_p:  631.66416<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  200.70657<br />Nuclei_Mean_Intensity_mean_RB1_p:  788.59859<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  316.33114<br />Nuclei_Mean_Intensity_mean_RB1_p:  951.42010<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  116.42857<br />Nuclei_Mean_Intensity_mean_RB1_p:  896.14011<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  135.96429<br />Nuclei_Mean_Intensity_mean_RB1_p:  745.46726<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  138.02249<br />Nuclei_Mean_Intensity_mean_RB1_p:  345.01840<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  132.84685<br />Nuclei_Mean_Intensity_mean_RB1_p: 1356.00676<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  243.45572<br />Nuclei_Mean_Intensity_mean_RB1_p:  597.97417<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  160.87275<br />Nuclei_Mean_Intensity_mean_RB1_p:  679.44952<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  136.04980<br />Nuclei_Mean_Intensity_mean_RB1_p:  582.52390<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  170.27079<br />Nuclei_Mean_Intensity_mean_RB1_p:  984.02772<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  156.13807<br />Nuclei_Mean_Intensity_mean_RB1_p:  627.01883<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  133.00000<br />Nuclei_Mean_Intensity_mean_RB1_p:  653.13411<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  166.63080<br />Nuclei_Mean_Intensity_mean_RB1_p:  676.08228<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  128.53460<br />Nuclei_Mean_Intensity_mean_RB1_p:  717.89965<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  446.92067<br />Nuclei_Mean_Intensity_mean_RB1_p:  885.29019<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   86.89683<br />Nuclei_Mean_Intensity_mean_RB1_p:  220.41005<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  112.90476<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.85714<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  139.05155<br />Nuclei_Mean_Intensity_mean_RB1_p: 1295.06186<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  130.07914<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.99041<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  534.00212<br />Nuclei_Mean_Intensity_mean_RB1_p:  677.53602<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  107.73709<br />Nuclei_Mean_Intensity_mean_RB1_p:  424.80986<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  229.45778<br />Nuclei_Mean_Intensity_mean_RB1_p: 1402.08667<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  162.79798<br />Nuclei_Mean_Intensity_mean_RB1_p:  771.73569<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  128.54481<br />Nuclei_Mean_Intensity_mean_RB1_p:  587.25835<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  211.28967<br />Nuclei_Mean_Intensity_mean_RB1_p:  424.26201<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  121.31670<br />Nuclei_Mean_Intensity_mean_RB1_p:  494.22994<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  139.03325<br />Nuclei_Mean_Intensity_mean_RB1_p:  937.37340<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  262.04386<br />Nuclei_Mean_Intensity_mean_RB1_p:  839.58421<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  117.69841<br />Nuclei_Mean_Intensity_mean_RB1_p:  413.90476<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  104.68308<br />Nuclei_Mean_Intensity_mean_RB1_p:  429.12000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  185.55714<br />Nuclei_Mean_Intensity_mean_RB1_p:  987.42143<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  222.13441<br />Nuclei_Mean_Intensity_mean_RB1_p:  845.17204<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  226.01456<br />Nuclei_Mean_Intensity_mean_RB1_p: 1564.70631<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  102.48824<br />Nuclei_Mean_Intensity_mean_RB1_p:  754.10294<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  107.92958<br />Nuclei_Mean_Intensity_mean_RB1_p:  510.77465<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  191.60351<br />Nuclei_Mean_Intensity_mean_RB1_p:  610.88070<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  229.21700<br />Nuclei_Mean_Intensity_mean_RB1_p:  739.33110<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   75.94083<br />Nuclei_Mean_Intensity_mean_RB1_p:  477.13314<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  295.08472<br />Nuclei_Mean_Intensity_mean_RB1_p:  834.02222<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  484.27122<br />Nuclei_Mean_Intensity_mean_RB1_p:  682.80627<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  101.04399<br />Nuclei_Mean_Intensity_mean_RB1_p:  506.38123<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   98.41503<br />Nuclei_Mean_Intensity_mean_RB1_p:  887.75163<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  104.06080<br />Nuclei_Mean_Intensity_mean_RB1_p:  511.54204<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   88.64214<br />Nuclei_Mean_Intensity_mean_RB1_p:  555.45819<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  108.92405<br />Nuclei_Mean_Intensity_mean_RB1_p:   28.04810<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  645.07538<br />Nuclei_Mean_Intensity_mean_RB1_p:  991.23451<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   75.77035<br />Nuclei_Mean_Intensity_mean_RB1_p:  672.47384<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  187.69778<br />Nuclei_Mean_Intensity_mean_RB1_p:  542.80667<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   90.00748<br />Nuclei_Mean_Intensity_mean_RB1_p: 1088.44888<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  146.15994<br />Nuclei_Mean_Intensity_mean_RB1_p:  753.59472<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  197.94987<br />Nuclei_Mean_Intensity_mean_RB1_p:  364.88391<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  161.68685<br />Nuclei_Mean_Intensity_mean_RB1_p:   21.89770<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  297.71670<br />Nuclei_Mean_Intensity_mean_RB1_p:  522.59865<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  278.87453<br />Nuclei_Mean_Intensity_mean_RB1_p:  644.67826<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  255.33072<br />Nuclei_Mean_Intensity_mean_RB1_p: 1595.12941<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   97.30488<br />Nuclei_Mean_Intensity_mean_RB1_p:  601.68902<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  210.20404<br />Nuclei_Mean_Intensity_mean_RB1_p:  770.86667<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  270.49902<br />Nuclei_Mean_Intensity_mean_RB1_p: 1126.88454<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  392.79331<br />Nuclei_Mean_Intensity_mean_RB1_p: 1091.07677<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  336.02198<br />Nuclei_Mean_Intensity_mean_RB1_p:   39.39780<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   97.71131<br />Nuclei_Mean_Intensity_mean_RB1_p:  974.43155<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  131.40470<br />Nuclei_Mean_Intensity_mean_RB1_p:  512.23760<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  168.42355<br />Nuclei_Mean_Intensity_mean_RB1_p: 1412.25413<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  155.89709<br />Nuclei_Mean_Intensity_mean_RB1_p:   30.16107<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  112.09417<br />Nuclei_Mean_Intensity_mean_RB1_p:  188.01121<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  492.77485<br />Nuclei_Mean_Intensity_mean_RB1_p:  922.13743<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  189.39910<br />Nuclei_Mean_Intensity_mean_RB1_p:  980.22197<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  110.72650<br />Nuclei_Mean_Intensity_mean_RB1_p:  803.20727<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  195.81942<br />Nuclei_Mean_Intensity_mean_RB1_p: 1345.29320<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  162.99123<br />Nuclei_Mean_Intensity_mean_RB1_p:  477.57310<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  105.20302<br />Nuclei_Mean_Intensity_mean_RB1_p:  170.66275<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  224.11776<br />Nuclei_Mean_Intensity_mean_RB1_p:  633.29907<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  132.09422<br />Nuclei_Mean_Intensity_mean_RB1_p:  477.93313<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  182.04031<br />Nuclei_Mean_Intensity_mean_RB1_p:  555.49136<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  177.70816<br />Nuclei_Mean_Intensity_mean_RB1_p:  891.58645<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  181.25448<br />Nuclei_Mean_Intensity_mean_RB1_p:  618.45520<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  101.48723<br />Nuclei_Mean_Intensity_mean_RB1_p: 1020.76424<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  219.32767<br />Nuclei_Mean_Intensity_mean_RB1_p:  430.15147<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  128.66667<br />Nuclei_Mean_Intensity_mean_RB1_p:   27.57239<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   89.82984<br />Nuclei_Mean_Intensity_mean_RB1_p:  418.16492<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  108.83455<br />Nuclei_Mean_Intensity_mean_RB1_p:  710.90511<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:   88.03333<br />Nuclei_Mean_Intensity_mean_RB1_p:  454.36667<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  155.42708<br />Nuclei_Mean_Intensity_mean_RB1_p:   24.83594<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  145.60582<br />Nuclei_Mean_Intensity_mean_RB1_p:  473.81260<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Nuclei_Mean_Intensity_mean_CCNA2:  148.32279<br />Nuclei_Mean_Intensity_mean_RB1_p:  510.00000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":48.9497716894977},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[5.30524212759685,10.6609264557702],"tickmode":"array","ticktext":["64","128","256","512","1024"],"tickvals":[6,7,8,9,10],"categoryorder":"array","categoryarray":["64","128","256","512","1024"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"Nuclei_Mean_Intensity_mean_CCNA2","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[4.05226753820856,11.8334171239498],"tickmode":"array","ticktext":["64","256","1024"],"tickvals":[6,8,10],"categoryorder":"array","categoryarray":["64","256","1024"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"Nuclei_Mean_Intensity_mean_RB1_p","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"19086f24fb0":{"x":{},"y":{},"text":{},"type":"scatter"}},"cur_data":"19086f24fb0","visdat":{"19086f24fb0":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-99e7de0bf63103d7be1b">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>