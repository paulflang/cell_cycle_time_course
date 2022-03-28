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
<div id="htmlwidget-819bb39a72f900d4387a" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-819bb39a72f900d4387a">{"x":{"data":[{"x":[34.925648,31.783971,25.770076,82.046492,20.270741,33.752718,22.044058,32.827603,36.813009,24.543012,32.996038,26.140297,25.020492,24.021541,33.459131,104.374637,145.32136,34.744378,33.251485,98.395919,80.678694,216.843615,88.708334,76.687043,30.461542,88.353542,22.398496,50.436798,64.779817,30.309394,93.255937,22.560624,19.516448,105.783955,21.691045,37.601985,28.067961,74.683483,72.045385,88.445853,72.331765,67.537803,96.788626,99.876268,99.361335,96.936919,126.443122,146.053498,87.634303,65.967169,61.632919,98.288515,76.50804,75.834401,68.869239,63.110763,70.009326,33.947589,106.871872,95.008442,154.565317,81.367038,46.139211,60.006368,54.561464,45.317687,103.118093,114.411839,74.066603,63.99179,95.872315,100.728361,113.030293,116.945559,76.695505,60.766868,83.565614,58.927741,83.964387,67.823871,62.876244,45.254408,79.4166,32.044917,66.880407,24.772228,71.216687,104.105593,29.803769,69.593906,107.404124,84.309634,70.825601,73.202547,62.152908,22.454624,75.344135,57.702019,104.201256,76.463643,76.266906,89.13457,92.042552,76.075139,106.515229,82.747097,29.412153,104.682479,137.39838,87.086631,67.000373,110.553868,59.032193,37.466406,9.156517,71.087352,84.293937,74.346005,81.809168,89.466626,89.337577,84.098406,65.953077,59.532857,132.760897,72.237819,176.34007,29.880726,105.411812,99.209584,94.764641,45.930144,92.593956,69.513338,34.940628,62.553971,98.688791,72.073466,51.246316,65.767942,54.468103,92.265599,63.716458,93.420399,89.511044,91.644575,77.181865,66.70485,80.09145,112.741865,59.444134,105.686638,114.704472,88.354459,27.626415,9.29753,69.543269,102.778195,77.119795,93.106911,136.387111,90.368599,114.762229,45.512127,85.027456,72.439211,82.771722,60.220818,86.085938,182.375229,87.403775,99.582261,102.463308,56.118565,100.815797,146.360003,83.029279,120.808285,129.913702,94.456248,82.873663,115.039776,88.075475,109.567682,122.37007,81.109707,17.577191,74.810158,74.080312,89.756216,129.942357,82.816159,96.826505,4.208834,33.877644,66.790469,29.201083,74.651116,25.249887,41.940918,37.596919,78.807645,29.593012,27.492167,30.176789,71.3504,29.531338,84.847201,99.486766,27.216683,30.601444,69.228368,60.134973,78.062714,88.594314,28.268188,12.830522,10.844704,12.253537,25.687203,8.576772,38.711583,73.714549,125.086937,95.100953,92.30025,30.378153,28.282831,124.293669,88.010291,74.314925,82.278424,81.228182,88.847531,70.304197,88.03603,53.931065,106.219761,69.588237,85.092713,67.060294,171.142629,91.585134,64.830089,71.707257,89.797442,126.130758,227.178439,85.321692,36.459971,33.234307,90.315395,66.810278,100.086971,50.220441,90.154118,60.137543,112.70084,64.938069,75.735545,70.62895,95.223391,126.09971,64.69301,94.720036,77.666994,99.476304,195.39762,123.265435,78.906164,68.726449,117.530802,179.675644,109.333357,101.156752,187.928875,114.753259,126.076275,62.064124,101.132127,58.332227,75.31306,97.287585,90.879137,93.175296,102.016705,80.749988,85.366251,73.174766,69.909592,106.970747,70.678825,4.742873,106.20733,91.858008,38.354349,72.058302,93.055384,56.188233,123.55226,4.530662,63.844111,43.690178,96.687254,66.909891,74.760998,84.385983,22.158454,94.358962,94.019586,25.895395,85.26173,180.017843,28.633199,24.651002,83.919729,58.592028,75.164042,88.716624,61.614964,108.39798,138.080056,13.035671,27.66682,85.39175,147.443366,97.82502,64.414137,30.810563,91.544263,94.757382,82.647206,73.612568,58.955107,98.187099,67.701041,67.47368,62.914203,81.269504,82.653443,98.839784,100.164547,68.433376,88.381686,79.86419,114.187306,116.145672,62.338182,83.267426,82.20284,92.765605,92.622514,44.148049,157.742519,125.235628,8.068961,111.017522,72.12496,76.320639,88.678548,143.349741,84.655706,104.580499,166.531074,85.169336,77.564998,79.660037,103.915019,67.939919,108.427325,124.397405,63.480806,78.763064,65.471736,93.024238,56.733713,87.073305,88.075713,54.850117,92.915754,79.621147,152.036253,122.486048,68.278917,37.598668,25.109259,91.611767,71.294048,102.218477,69.217753,72.321194,29.443897,83.114412,78.155403,63.637809,90.71023,66.232138,210.649824,132.709911,88.715239,64.034989,64.512323,100.039338,27.972019,26.983931,83.864355,86.143788,74.159804,89.537519,123.272796,89.062064,93.122212,142.146803,146.977457,92.469424,82.78495,117.413657,108.317264,94.436809,164.567364,101.558582,72.305605,88.604851,44.234808,78.09186,67.519385,91.875087,121.85573,62.206524,98.57479,28.139315,103.368072,102.832672,28.567294,151.016913,81.041833,87.542597,128.551283,63.854565,66.656741,194.064099,47.479082,57.33394,41.308374,59.833458,52.097559,95.799376,110.294377,68.350374,102.739852,96.127321,49.150436,74.534336,78.47424,80.279982,101.112371,92.410043,66.935807,84.229375,61.252888,91.794076,71.261559,76.448249,49.965202,59.830689,89.320949,83.084805,95.442201,100.002818,80.380039,97.279388,79.39038,82.288145,82.825897,105.594885,67.388908,100.259252,91.447615,68.360591,77.915338,87.65888,87.716974,51.405273,72.333963,94.197751,72.573149,89.029095,91.859977,97.311715,96.824248,96.744656,58.884638,84.890164,68.235717,86.325703,123.592538,87.176235,107.490794,111.402398,85.722016,107.312489,108.737178,58.219201,217.985584,182.98918,92.609334,80.937594,42.993377,102.318012,117.25691,88.510402,83.891359,76.269929,97.883094,90.416361,70.790685,5.215362,73.896318,76.86568,111.014993,77.178276,88.871935,160.413812,82.513109,74.233496,72.933215,84.758216,88.907899,82.891973,57.207916,116.078353,97.437097,5.542593,102.473916,83.004652,132.961583,102.753096,106.941302,150.043683,77.372322,95.311185,74.213636,94.716778,88.62738,116.675518,119.403674,37.517871,120.147448,81.721456,76.304792,101.415615,109.024277,98.97784,35.551712,82.985519,38.933753,167.450741,31.788619,82.091145,122.764918,126.987458,112.971407,77.435371,83.736905,59.301735,89.236683,63.905312,38.928609,64.843345,92.62376,80.373741,92.081018,72.429642,61.795027,91.27373,78.091597,62.228022,93.303585,66.582284,116.989735,70.986944,90.458378,138.089003,70.401412,94.798451,85.465743,60.74723,65.757614,77.895534,92.620092,75.380532,4.318667,76.660778,88.995528,92.946346,87.303186,90.209404,95.965855,89.714222,132.319686,65.204215,58.242599,46.021336,64.333638,84.719761,88.679156,105.66289,80.966331,42.914187,210.53506,83.340697,85.114541,147.172382,133.511967,79.07272,69.30371,83.08049,81.860113,98.075449,68.79275,64.19465,128.945336,83.62631,33.069163,73.557633,95.816902,80.307697,164.251818,86.544424,64.629246,107.129745,66.132194,79.538189,105.005308,32.535359,67.680593,146.122198,84.426113,99.573422,129.575191,94.671101,70.288314,69.444597,72.807255,86.459281,133.26949,211.906336,134.734578,66.191845,58.846316,158.269098,108.93282,103.752843,92.223095,17.517962,5.556314,58.201802,70.185359,65.383511,42.491995,37.045515,72.023535,50.365249,100.791965,107.815851,127.582925,56.422631,21.272986,90.581419,84.262498,91.012523,85.324687,124.466217,73.507311,109.193075,76.363071,93.967179,59.357059,56.634975,70.114066,108.420965,100.374501,78.537984,50.168885,66.866732,61.412251,60.398334,48.632959,82.346996,95.271958,96.458461,89.843999,161.826606,101.134908,57.667377,96.767759,77.685979,61.435992,137.177886,113.397066,100.01463,119.622466,74.689753,83.048821,88.607716,135.032809,25.232275,117.804735,80.722388,116.819251,68.155354,81.92366,77.345968,66.862071,101.47891,66.589417,61.362834,87.575,93.999702,115.409292,109.484569,129.212577,95.672395,96.929323,69.335458,67.154939,127.435587,94.709445,71.050232,49.496058,55.348347,99.670964,151.917567,98.464519,80.451962,67.557178,67.794203,92.69189,83.903722,55.497125,59.389004,73.285135,68.580179,96.563536,85.583576,83.911042,96.038202,100.377229,113.0881,36.724176,12.081709,75.979598,119.765226,53.592651,84.538036,52.474916,54.928173,115.073074,107.527555,92.339767,94.645559,85.779844,96.071507,77.780878,73.538108,94.642218,73.940414,73.204652,94.277239,72.296552,148.472819,105.821727,149.65604,127.017437,81.743531,103.154183,90.186227,153.897179,72.481395,106.007178,109.028258,79.882388,26.024659,96.372898,91.865481,80.088578,59.654611,110.225765,102.124847,105.618858,109.47002,93.97485,75.344839,115.31777,109.768933,74.902391,83.708079,106.295088,155.835086,50.96736,118.994453,120.243263,47.47706,105.21161,65.838881,89.82243,69.864035,102.937734,129.824585,93.487424,5.540493,62.946041,78.82946,105.40218,80.207395,67.061872,119.850546,152.535029,132.083525,75.861179,70.26483,102.095853,60.544005,78.130445,127.57083,75.525089,100.256804,94.028383,118.594594,85.163626,105.732075,91.275506,110.281094,36.489905,145.449048,105.688412,64.970756,76.669006,63.229083,143.017648,98.111308,48.245515,31.878379,204.404786,89.985535,65.444762,60.738209,158.771314,69.443446,113.100473,128.211327,62.227764,86.051071,80.071532,12.465912,92.538805,65.905033,56.069304,75.538681,97.472647,77.483832,104.593991,180.899733,76.062519,81.918305,65.445748,98.520738,75.092971,64.821613,121.270596,87.575215,63.062166,99.220207,84.277978,96.323582,72.664399,96.902759,103.203451,102.474278,131.764759,109.965704,73.796963,102.094692,73.557548,75.541972,79.34471,72.411715,67.470712,100.867388,70.055527,72.154649,85.94372,103.390653,73.911163,69.779181,66.121599,101.934405,45.547961,69.332915,93.295672,49.06657,67.660041,78.442415,47.916294,88.726467,216.276556,59.31043,133.940995,95.711036,120.706516,86.281184,51.005668,125.189341,83.235532,78.24666,104.78372,70.048191,91.712441,70.238884,102.04341,76.122255,110.903565,90.19123,88.912828,126.168848,125.053965,176.805903,104.777819,70.762619,62.799878,68.364236,203.195095,112.469796,92.367706,31.495477,105.495119,73.100593,33.783153,50.930818,121.529054,80.599487,110.954928,80.474684,86.308028],"y":[1,1,1,2.70091324200913,1,1,1,1,1,1,1,1,1,1,1,10.6885245901639,4.89569990850869,1,1,7.09815950920245,6.47058823529412,7.39774330042313,2.45240532241556,4.1031941031941,1,4.62714776632302,1,1,4.57487922705314,1,3.53358208955224,1,1,9.48118811881188,1,2.72222222222222,1,4.12293853073463,6.88760806916426,8.55429864253394,4.54464285714286,4.05,6.38963210702341,5.44897959183673,2.70385395537525,7.02380952380952,8.96707818930041,4.40829694323144,3.58578199052133,5.49693251533742,2.52336448598131,7.66309012875536,4.11730769230769,5.48186528497409,4.75854214123007,4.38815789473684,5.34762979683973,1.4336569579288,3.63419913419913,7.59913793103448,6.02150537634409,5.24742268041237,4.14912280701754,4,3.39877300613497,3.896875,4.51577287066246,6.51510574018127,6.46419098143236,5.89945652173913,6.00932400932401,7.84577922077922,4.99117647058824,5.17612809315866,7.13057324840764,6.81754385964912,10.2131578947368,2.99290780141844,3.99548532731377,6.17796610169492,5.89235127478754,3.39285714285714,9.43070362473348,1,2.84728340675477,1.57954545454545,4.17922077922078,6.24181626187962,1,4.64229765013055,7.02586206896552,6.02805611222445,6.41791044776119,6.04439252336449,4.44657534246575,1.5985401459854,5.55981941309255,5.18207282913165,3.23127753303965,6.43594306049822,6.62407862407862,4.88811188811189,5.61961722488038,5.48291571753986,2.7773851590106,5.80465949820789,1,7.3349593495935,10.5224171539961,4.708984375,5.61835748792271,7.01183431952663,4.73893805309735,1,1,5.19430051813471,6.27455919395466,4.91192660550459,5.52097130242826,5.98441558441558,6.77483443708609,5.89294403892944,4.61560693641619,5.87743732590529,9.55393586005831,7.23140495867769,7.07631160572337,1,7.48333333333333,5.9750566893424,4.34009360374415,3.34955752212389,4.5344262295082,3.42291666666667,1,5.31454005934718,4.4576,3.47657512116317,2.5699481865285,8.0377358490566,4.1793893129771,6.57427937915743,5.54123711340206,4.27391304347826,6.81056466302368,6.36175710594315,7.40116279069767,5.54766031195841,5.38135593220339,5.74676524953789,6.21153846153846,6.22393162393162,5.53789731051345,6.48927038626609,4.57142857142857,2,6.20293398533007,5.30841121495327,4.41014799154334,5.41187050359712,8.63189269746647,6.02469135802469,6.39253731343284,1.56372549019608,5.96407185628743,4.20744680851064,6.01589825119237,4.49512195121951,7.3508064516129,6.78391959798995,5.14807302231237,6.35294117647059,5.8014888337469,3.93632075471698,9.43213296398892,9.73392461197339,6.75,3.11796733212341,6.90121951219512,5.9533527696793,5.30944625407166,10.7867803837953,4.44171779141104,5.99344692005242,6.89079229122056,8.29820627802691,1.44615384615385,6.699203187251,5.99043062200957,7.11923076923077,6.13565891472868,6.80348258706468,6.11021505376344,1,1,6.77133105802048,1,5.9249530956848,1,1,1.83967391304348,5.46025878003697,1,1,1,4.88571428571429,1,2.76190476190476,8.37637362637363,1.03214285714286,1.01682242990654,6.29607250755287,4.03235294117647,6.93786982248521,6.54264972776769,1,1,1,1,3.40677966101695,1,1,4.77647058823529,5.66629834254144,5.37482710926694,5.3309481216458,1,1.00689655172414,6.41706161137441,4.84431977559607,6.16420664206642,7.00576923076923,7.78758169934641,7.42553191489362,7.8695652173913,4.89022556390977,5.06104651162791,5.85538461538462,6.73316708229426,6.4581589958159,5.41624365482233,10.2694444444444,6.30535714285714,4.84848484848485,5.22968197879859,6.43106796116505,7.70232558139535,9.77453027139875,4.86666666666667,2.52702702702703,1,6.19735099337748,4.5082304526749,7.38188277087034,5.34154929577465,5.23801369863014,4.89333333333333,6.44428969359331,6.74641148325359,7.74142480211082,4.15094339622642,5.40443686006826,7.27551020408163,6.66028708133971,5.5472263868066,6.50966608084359,6.13853503184713,15.8702163061564,6.5103305785124,7.57411764705882,4.66379310344828,6.54712643678161,7.38123167155425,6.70436507936508,4.48614609571788,8.49718151071026,7.34860557768924,12.3378378378378,4.7345971563981,7.51004016064257,3.3864118895966,7.95478723404255,5.39180327868852,7.06212424849699,5.44220183486239,4.91707317073171,6.86799276672694,7.986,4.86706349206349,3.71393643031785,8.22122571001495,5.05924170616114,1,6.66271186440678,8.6053412462908,6.47899159663866,4.14834205933682,25.5431034482759,15.0526315789474,5.97040169133192,1,3.74960380348653,4.04102564102564,9.80535279805353,8.6309963099631,5.99759615384615,10.5326086956522,15.4,18.7169811320755,296.214285714286,1.23857868020305,32.2952380952381,56.4509803921569,1.91099476439791,22.3076923076923,11.1208333333333,5.45238095238095,6.62095238095238,7.31391585760518,19.5083333333333,5.48109243697479,22.4713656387665,1.65217391304348,25.5,5.9951690821256,12.4618937644342,4.7056856187291,5.02910052910053,1,6.88674033149171,4.96909492273731,5.57805907172996,5.8983451536643,5.04136253041363,7.24950884086444,5.41711229946524,5.20955882352941,6.11413043478261,5.71760154738878,5.84735812133072,5.0613810741688,6.31628787878788,6.06586826347305,6.27194492254733,5.53833605220228,6.92494481236203,4.32439926062847,5.78532608695652,5.57172131147541,4.86046511627907,6.52873563218391,8.53133514986376,1.97963800904977,6.69094138543517,5.48877374784111,1,4.6,6.26741573033708,6.46406570841889,6.61467889908257,5.96398891966759,4.10578842315369,5.62199312714777,7.71428571428571,5.25518341307815,5.90144230769231,7.55379746835443,7.05357142857143,3.75675675675676,6.37317073170732,5.53341902313625,6.21502590673575,5.03758169934641,3.62939958592133,5.23745173745174,5.55714285714286,5.98193760262726,7.40659340659341,5.89510489510489,5.57121661721068,5.90407673860911,5.73177842565598,4.28353658536585,4.64571428571429,2.80921052631579,1.49242424242424,6.189852700491,3.44,5.4968152866242,4.03521126760563,3.83333333333333,1.44360902255639,6.38349514563107,5.38766519823789,4.45983935742972,7.60204081632653,2.85897435897436,13.3832116788321,7.446735395189,4.72287390029325,8.06539509536785,5.25077399380805,6.17995444191344,1.14576271186441,2.47272727272727,5.46231884057971,5.20039682539683,4.81918819188192,5.93103448275862,6.04093567251462,6.38803418803419,5.53745541022592,7.16402116402116,9.16734693877551,5.2781954887218,6.13140604467806,5.38503401360544,5.40549828178694,4.62475442043222,8.35670731707317,4.23185840707965,4.01594896331738,6.59824046920821,1.48484848484848,5.67172675521822,5.60989010989011,5.80687397708674,7.14627994955864,5.17523364485981,6.64884393063584,1,6.49689440993789,5.97011952191235,1.53731343283582,11.9596412556054,4.84586466165414,7.2962962962963,11.1400709219858,4.90956072351421,6.63836477987421,12.6089171974522,2.69296375266525,4.05464480874317,3.19283746556474,3.73641304347826,3.33271719038817,5.1839863713799,7.149377593361,7.5544794188862,5.84060402684564,4.48916967509025,4.21875,4.82075471698113,7.37905236907731,5.2625,7.75892857142857,9.40845070422535,5.89442815249267,6.94475138121547,5.36885245901639,7.98133333333333,6.56650246305419,6.7010752688172,4.8558282208589,5.04424778761062,4.84401114206128,6.59459459459459,7.33333333333333,5.49740034662045,6.15274949083503,6.83411580594679,4.35943775100402,6.57372654155496,6.36080178173719,7.32370820668693,4.0599173553719,5.92792792792793,5.10803324099723,5.00244498777506,6.83969465648855,5.54901960784314,5.93168880455408,4.03225806451613,5.43198090692124,9.30215827338129,5.30888030888031,6.11928934010152,5.38194444444444,5.60232945091514,6.83847980997625,7.46792452830189,4.9645390070922,4.14669051878354,3.6213921901528,4.41285956006768,5.34825870646766,4.76702508960573,5.15822784810127,7.28035982008995,5.58649093904448,4.40861618798956,8.88308457711443,4.84375,6.32142857142857,17.2252873563218,8.68539325842697,6.065,1.30081300813008,6.36712749615975,6.04658901830283,6.58862144420131,7.25116279069767,3.96822995461422,7.16363636363636,6.17133443163097,8.10924369747899,1,6.03007518796993,4.43809523809524,6.68217054263566,5.8871473354232,5.34602076124567,11.9630606860158,5.155,7.45911949685535,6.03621169916435,6.37212863705972,9.10055865921788,4.94117647058824,4.29925650557621,5.39498432601881,14.1551724137931,1,8.77713178294574,5.40885860306644,7.89388489208633,4.67493112947658,8.46743295019157,6.93554006968641,6.06775700934579,16.9225092250923,5.50558213716108,9.08939708939709,7.94736842105263,5.81093394077449,5.6528384279476,1,8.11481481481482,6.23983739837398,6.69762845849802,5.56344410876133,4.47208931419458,4.40096618357488,1.82251082251082,5.91533180778032,1.8294930875576,10.2264957264957,1,8.51928020565553,7.80650994575045,6.75558659217877,4.96560196560197,5.52057245080501,6.39393939393939,3.42738589211618,6.9620253164557,6.28918322295806,4.60576923076923,4.50331125827815,7.03862660944206,8.38912133891213,4.7262969588551,6.29479768786127,5.85507246376812,6.05836575875486,5.98584905660377,4.88529411764706,6.7059925093633,5.17439293598234,6.81222707423581,4.89443378119002,5.57917261055635,5.52446183953033,6.38120104438642,7.235,6.88814317673378,5.27565982404692,7.53387533875339,5.34123222748815,6.6016713091922,6.86075949367089,1,4.575,6.12219451371571,9.03157894736842,5.69849931787176,9.71490280777538,5.75636363636364,6.45514223194748,9.86599664991625,7.11007025761124,5.8992443324937,4.27898550724638,7.16480446927374,10.3316953316953,6.07363420427553,3.93119266055046,4.46718146718147,3.23985239852399,6.48843930635838,7.71571072319202,9.13272311212815,7.96859504132231,4.48858447488584,8.00617283950617,6.09090909090909,7.33333333333333,5.67624521072797,6.29597197898424,3.45616641901932,5.56401384083045,8.11842105263158,5.8957528957529,1,5.92560175054705,5.23318385650224,5.4930966469428,6.21644295302013,5.14231499051233,6.59911894273128,6.05743243243243,6.97428571428571,5.79089026915114,7.71851851851852,1,6.90384615384615,3.25849056603774,6.1969696969697,6.20357941834452,5.16720779220779,5.59392789373814,3.4989604989605,4.9847972972973,2.80082135523614,6.17277486910995,9.19093406593407,13.0780141843972,6.58135593220339,5.23650385604113,5.57777777777778,18.2046783625731,15.9050632911392,5.93896713615023,6.43445692883895,0.856353591160221,1,6.25549450549451,5.3096926713948,5.52,1.56792452830189,4.01546391752577,3.81325301204819,4.16849816849817,6.17234468937876,7.91320754716981,5.15188172043011,5.19718309859155,1.47311827956989,7.9727047146402,7.35381355932203,6.10746268656716,5.06885758998435,8.04611650485437,5.45971563981043,7.87628865979381,5.77643504531722,5.01063829787234,5.36923076923077,4.79005524861878,3.25151515151515,5.02088167053364,5.06151832460733,5.76744186046512,4.57432432432432,6.14210526315789,5.08977556109726,5.48347107438016,5.32394366197183,7.30922693266833,5.87339449541284,7.63385826771654,6.34782608695652,10.5788461538462,6.28571428571429,6.27054794520548,10,4.23145400593472,5.75392670157068,9.64168190127971,5.18627450980392,6.6008316008316,6.88556338028169,5.60994764397906,4.30517711171662,6.55047619047619,6.65720524017467,1.26905829596413,10.5768500948767,5.37275985663082,6.66619718309859,7.2,6.59273422562141,5.24822695035461,3.4009900990099,11.9765807962529,7.3232044198895,5.36238532110092,6.144,7.22011385199241,6.61596009975062,5.71990171990172,7.28458498023715,5.60051546391753,7.04148148148148,6.14081145584726,4.23474178403756,7.9553450608931,5.662100456621,6.13062098501071,5.021875,5.00313479623824,5.13915857605178,8.1843220338983,4.83790523690773,9.0479797979798,4.38059701492537,7.41486068111455,4.92324561403509,8.48988764044944,3.77873563218391,5.04076086956522,6.53266331658291,5.95666666666667,6.30737704918033,5.93373493975904,6.62650602409639,8.60330578512397,10.6984732824427,6.77252252252252,1.57881136950904,2.2,5.50151057401813,6.66259541984733,3.56875,9.59459459459459,3.43490304709141,3.72435897435897,6.30777903043968,4.95910780669145,6.08651399491094,7.06649616368286,5.84257871064468,5.89889415481833,6.70801033591731,5.71561338289963,6.54152823920266,4.55701754385965,6.92813141683778,7.88387096774194,6.54521963824289,6.63715529753266,7.44715447154472,8.2253829321663,9.99823943661972,5.45557655954631,4.97039473684211,4.81698113207547,9.99132947976879,8.40369393139842,8.24455205811138,6.30570902394107,6.11992263056093,1,4.62062937062937,4.44201680672269,7.31197771587744,7.11635220125786,4.08552631578947,7.35232383808096,9.42943548387097,6.45492957746479,6.48198198198198,5.64871794871795,5.99476439790576,5.84413309982487,6.82096069868996,3.78694817658349,6.9305912596401,7.05263157894737,4.53333333333333,6.25508607198748,6.64848484848485,4.09454545454545,5.86714542190305,5.79336734693878,6.90022172949002,5.14588235294118,9.47520661157025,7.25482625482625,6.93248175182482,1,5.62662337662338,6.51905626134301,5.5,6.64779874213836,5.52843601895735,6.91603053435114,7.34730538922156,3.98478561549101,5.28287841191067,6.06632653061224,4.99509803921569,5.67813267813268,6.88817891373802,7.4175654853621,7.49060542797495,7.28650137741047,9.43761996161228,8.37769080234834,9.58690176322418,7.14373716632443,7.73489278752437,6.5,3.03793103448276,5.67632850241546,4.14574898785425,7.00727272727273,4.44993662864385,4.09497206703911,4.60248447204969,4.14516129032258,5.44483985765125,2.25757575757576,5.38191881918819,5.83159722222222,4.421875,5.34866828087167,5.22284644194757,5.80361173814898,5.14564564564565,6.49802371541502,5.41428571428571,5.04511278195489,4.5962441314554,1.43181818181818,5.1004942339374,6.79945054945055,4.52083333333333,5.60122699386503,6.19594594594595,5.40590405904059,6.6210235131397,6.51792828685259,5.94456289978678,5.70083682008368,6.64431486880467,7.74683544303798,6.30795847750865,4.28810020876827,7.20899470899471,6.60714285714286,5.69072164948454,5.48681055155875,5.62076271186441,8.24882629107981,5.81555555555556,5.47811447811448,6.4780316344464,7.10334788937409,7.51843817787419,7.11764705882353,4.33508771929825,6.06613756613757,6.44,6.10238095238095,6.75985663082437,4.28398058252427,6.77058823529412,4.64507042253521,4.37894736842105,5.68680089485459,5.0414201183432,7.86527777777778,3.57933579335793,4.61290322580645,5.2516339869281,4.66623544631307,3.7123745819398,5.67341772151899,6.51256281407035,4.05232558139535,6.83777777777778,6.66944908180301,3.37905236907731,6.61801242236025,11.8254156769596,6.31398416886544,8.46137787056368,7.08768971332209,8.27950310559006,4.29411764705882,4.17073170731707,5.85858585858586,4.86497064579256,4.67322834645669,6.62637362637363,6.69940476190476,6.47519582245431,7.3202479338843,10.2460850111857,7.18834080717489,3.7719298245614,6.61883408071749,9.22863247863248,5.67378640776699,8.74122807017544,9.04697986577181,6.3607476635514,7.39817629179331,7.46666666666667,4.56046065259117,6.65421853388658,6.03405017921147,5.54420432220039,2.63235294117647,7.92272024729521,7.68013468013468,3.49068322981366,4.01570680628272,7.56934306569343,6.34166666666667,8.49739583333333,5.62358642972536,7.30379746835443],"text":["Mean_Morphology_Major_Axis_Length:  34.925648<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.783971<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.770076<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  82.046492<br />segmentation_metric:   2.7009132<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  20.270741<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.752718<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.044058<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.827603<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.813009<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.543012<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.996038<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.140297<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.020492<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.021541<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.459131<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 104.374637<br />segmentation_metric:  10.6885246<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 145.321360<br />segmentation_metric:   4.8956999<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  34.744378<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.251485<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  98.395919<br />segmentation_metric:   7.0981595<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  80.678694<br />segmentation_metric:   6.4705882<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 216.843615<br />segmentation_metric:   7.3977433<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  88.708334<br />segmentation_metric:   2.4524053<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  76.687043<br />segmentation_metric:   4.1031941<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.461542<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  88.353542<br />segmentation_metric:   4.6271478<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.398496<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  50.436798<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  64.779817<br />segmentation_metric:   4.5748792<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.309394<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  93.255937<br />segmentation_metric:   3.5335821<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.560624<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  19.516448<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 105.783955<br />segmentation_metric:   9.4811881<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  21.691045<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  37.601985<br />segmentation_metric:   2.7222222<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.067961<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.683483<br />segmentation_metric:   4.1229385<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  72.045385<br />segmentation_metric:   6.8876081<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  88.445853<br />segmentation_metric:   8.5542986<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  72.331765<br />segmentation_metric:   4.5446429<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  67.537803<br />segmentation_metric:   4.0500000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  96.788626<br />segmentation_metric:   6.3896321<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  99.876268<br />segmentation_metric:   5.4489796<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  99.361335<br />segmentation_metric:   2.7038540<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  96.936919<br />segmentation_metric:   7.0238095<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 126.443122<br />segmentation_metric:   8.9670782<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 146.053498<br />segmentation_metric:   4.4082969<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  87.634303<br />segmentation_metric:   3.5857820<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.967169<br />segmentation_metric:   5.4969325<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.632919<br />segmentation_metric:   2.5233645<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  98.288515<br />segmentation_metric:   7.6630901<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  76.508040<br />segmentation_metric:   4.1173077<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  75.834401<br />segmentation_metric:   5.4818653<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  68.869239<br />segmentation_metric:   4.7585421<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  63.110763<br />segmentation_metric:   4.3881579<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  70.009326<br />segmentation_metric:   5.3476298<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.947589<br />segmentation_metric:   1.4336570<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 106.871872<br />segmentation_metric:   3.6341991<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.008442<br />segmentation_metric:   7.5991379<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 154.565317<br />segmentation_metric:   6.0215054<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  81.367038<br />segmentation_metric:   5.2474227<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  46.139211<br />segmentation_metric:   4.1491228<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  60.006368<br />segmentation_metric:   4.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  54.561464<br />segmentation_metric:   3.3987730<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  45.317687<br />segmentation_metric:   3.8968750<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 103.118093<br />segmentation_metric:   4.5157729<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 114.411839<br />segmentation_metric:   6.5151057<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.066603<br />segmentation_metric:   6.4641910<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  63.991790<br />segmentation_metric:   5.8994565<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.872315<br />segmentation_metric:   6.0093240<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 100.728361<br />segmentation_metric:   7.8457792<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 113.030293<br />segmentation_metric:   4.9911765<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 116.945559<br />segmentation_metric:   5.1761281<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  76.695505<br />segmentation_metric:   7.1305732<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  60.766868<br />segmentation_metric:   6.8175439<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  83.565614<br />segmentation_metric:  10.2131579<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  58.927741<br />segmentation_metric:   2.9929078<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  83.964387<br />segmentation_metric:   3.9954853<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  67.823871<br />segmentation_metric:   6.1779661<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  62.876244<br />segmentation_metric:   5.8923513<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  45.254408<br />segmentation_metric:   3.3928571<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  79.416600<br />segmentation_metric:   9.4307036<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.044917<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  66.880407<br />segmentation_metric:   2.8472834<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.772228<br />segmentation_metric:   1.5795455<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.216687<br />segmentation_metric:   4.1792208<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 104.105593<br />segmentation_metric:   6.2418163<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.803769<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  69.593906<br />segmentation_metric:   4.6422977<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 107.404124<br />segmentation_metric:   7.0258621<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  84.309634<br />segmentation_metric:   6.0280561<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  70.825601<br />segmentation_metric:   6.4179104<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.202547<br />segmentation_metric:   6.0443925<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  62.152908<br />segmentation_metric:   4.4465753<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  22.454624<br />segmentation_metric:   1.5985401<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  75.344135<br />segmentation_metric:   5.5598194<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  57.702019<br />segmentation_metric:   5.1820728<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 104.201256<br />segmentation_metric:   3.2312775<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.463643<br />segmentation_metric:   6.4359431<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.266906<br />segmentation_metric:   6.6240786<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.134570<br />segmentation_metric:   4.8881119<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.042552<br />segmentation_metric:   5.6196172<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.075139<br />segmentation_metric:   5.4829157<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 106.515229<br />segmentation_metric:   2.7773852<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.747097<br />segmentation_metric:   5.8046595<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.412153<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 104.682479<br />segmentation_metric:   7.3349593<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 137.398380<br />segmentation_metric:  10.5224172<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  87.086631<br />segmentation_metric:   4.7089844<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  67.000373<br />segmentation_metric:   5.6183575<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 110.553868<br />segmentation_metric:   7.0118343<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.032193<br />segmentation_metric:   4.7389381<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.466406<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   9.156517<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.087352<br />segmentation_metric:   5.1943005<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  84.293937<br />segmentation_metric:   6.2745592<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.346005<br />segmentation_metric:   4.9119266<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.809168<br />segmentation_metric:   5.5209713<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.466626<br />segmentation_metric:   5.9844156<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.337577<br />segmentation_metric:   6.7748344<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  84.098406<br />segmentation_metric:   5.8929440<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  65.953077<br />segmentation_metric:   4.6156069<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.532857<br />segmentation_metric:   5.8774373<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 132.760897<br />segmentation_metric:   9.5539359<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.237819<br />segmentation_metric:   7.2314050<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 176.340070<br />segmentation_metric:   7.0763116<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.880726<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 105.411812<br />segmentation_metric:   7.4833333<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  99.209584<br />segmentation_metric:   5.9750567<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.764641<br />segmentation_metric:   4.3400936<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  45.930144<br />segmentation_metric:   3.3495575<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.593956<br />segmentation_metric:   4.5344262<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.513338<br />segmentation_metric:   3.4229167<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.940628<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  62.553971<br />segmentation_metric:   5.3145401<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  98.688791<br />segmentation_metric:   4.4576000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.073466<br />segmentation_metric:   3.4765751<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  51.246316<br />segmentation_metric:   2.5699482<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  65.767942<br />segmentation_metric:   8.0377358<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  54.468103<br />segmentation_metric:   4.1793893<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.265599<br />segmentation_metric:   6.5742794<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  63.716458<br />segmentation_metric:   5.5412371<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  93.420399<br />segmentation_metric:   4.2739130<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.511044<br />segmentation_metric:   6.8105647<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  91.644575<br />segmentation_metric:   6.3617571<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.181865<br />segmentation_metric:   7.4011628<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  66.704850<br />segmentation_metric:   5.5476603<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  80.091450<br />segmentation_metric:   5.3813559<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 112.741865<br />segmentation_metric:   5.7467652<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.444134<br />segmentation_metric:   6.2115385<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 105.686638<br />segmentation_metric:   6.2239316<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 114.704472<br />segmentation_metric:   5.5378973<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.354459<br />segmentation_metric:   6.4892704<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.626415<br />segmentation_metric:   4.5714286<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   9.297530<br />segmentation_metric:   2.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.543269<br />segmentation_metric:   6.2029340<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 102.778195<br />segmentation_metric:   5.3084112<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.119795<br />segmentation_metric:   4.4101480<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  93.106911<br />segmentation_metric:   5.4118705<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 136.387111<br />segmentation_metric:   8.6318927<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  90.368599<br />segmentation_metric:   6.0246914<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 114.762229<br />segmentation_metric:   6.3925373<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  45.512127<br />segmentation_metric:   1.5637255<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  85.027456<br />segmentation_metric:   5.9640719<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.439211<br />segmentation_metric:   4.2074468<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.771722<br />segmentation_metric:   6.0158983<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  60.220818<br />segmentation_metric:   4.4951220<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  86.085938<br />segmentation_metric:   7.3508065<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 182.375229<br />segmentation_metric:   6.7839196<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  87.403775<br />segmentation_metric:   5.1480730<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  99.582261<br />segmentation_metric:   6.3529412<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 102.463308<br />segmentation_metric:   5.8014888<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  56.118565<br />segmentation_metric:   3.9363208<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 100.815797<br />segmentation_metric:   9.4321330<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 146.360003<br />segmentation_metric:   9.7339246<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  83.029279<br />segmentation_metric:   6.7500000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 120.808285<br />segmentation_metric:   3.1179673<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 129.913702<br />segmentation_metric:   6.9012195<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.456248<br />segmentation_metric:   5.9533528<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.873663<br />segmentation_metric:   5.3094463<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 115.039776<br />segmentation_metric:  10.7867804<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.075475<br />segmentation_metric:   4.4417178<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 109.567682<br />segmentation_metric:   5.9934469<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 122.370070<br />segmentation_metric:   6.8907923<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.109707<br />segmentation_metric:   8.2982063<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  17.577191<br />segmentation_metric:   1.4461538<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.810158<br />segmentation_metric:   6.6992032<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.080312<br />segmentation_metric:   5.9904306<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.756216<br />segmentation_metric:   7.1192308<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 129.942357<br />segmentation_metric:   6.1356589<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.816159<br />segmentation_metric:   6.8034826<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  96.826505<br />segmentation_metric:   6.1102151<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   4.208834<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.877644<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  66.790469<br />segmentation_metric:   6.7713311<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.201083<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.651116<br />segmentation_metric:   5.9249531<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.249887<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  41.940918<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.596919<br />segmentation_metric:   1.8396739<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.807645<br />segmentation_metric:   5.4602588<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.593012<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.492167<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.176789<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.350400<br />segmentation_metric:   4.8857143<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.531338<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  84.847201<br />segmentation_metric:   2.7619048<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  99.486766<br />segmentation_metric:   8.3763736<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.216683<br />segmentation_metric:   1.0321429<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.601444<br />segmentation_metric:   1.0168224<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.228368<br />segmentation_metric:   6.2960725<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  60.134973<br />segmentation_metric:   4.0323529<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.062714<br />segmentation_metric:   6.9378698<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.594314<br />segmentation_metric:   6.5426497<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.268188<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  12.830522<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  10.844704<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  12.253537<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.687203<br />segmentation_metric:   3.4067797<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   8.576772<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  38.711583<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  73.714549<br />segmentation_metric:   4.7764706<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 125.086937<br />segmentation_metric:   5.6662983<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  95.100953<br />segmentation_metric:   5.3748271<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.300250<br />segmentation_metric:   5.3309481<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.378153<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.282831<br />segmentation_metric:   1.0068966<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 124.293669<br />segmentation_metric:   6.4170616<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.010291<br />segmentation_metric:   4.8443198<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.314925<br />segmentation_metric:   6.1642066<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.278424<br />segmentation_metric:   7.0057692<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.228182<br />segmentation_metric:   7.7875817<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.847531<br />segmentation_metric:   7.4255319<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  70.304197<br />segmentation_metric:   7.8695652<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.036030<br />segmentation_metric:   4.8902256<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  53.931065<br />segmentation_metric:   5.0610465<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.219761<br />segmentation_metric:   5.8553846<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.588237<br />segmentation_metric:   6.7331671<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.092713<br />segmentation_metric:   6.4581590<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.060294<br />segmentation_metric:   5.4162437<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 171.142629<br />segmentation_metric:  10.2694444<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.585134<br />segmentation_metric:   6.3053571<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.830089<br />segmentation_metric:   4.8484848<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  71.707257<br />segmentation_metric:   5.2296820<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.797442<br />segmentation_metric:   6.4310680<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.130758<br />segmentation_metric:   7.7023256<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 227.178439<br />segmentation_metric:   9.7745303<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.321692<br />segmentation_metric:   4.8666667<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  36.459971<br />segmentation_metric:   2.5270270<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  33.234307<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.315395<br />segmentation_metric:   6.1973510<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  66.810278<br />segmentation_metric:   4.5082305<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.086971<br />segmentation_metric:   7.3818828<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  50.220441<br />segmentation_metric:   5.3415493<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.154118<br />segmentation_metric:   5.2380137<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  60.137543<br />segmentation_metric:   4.8933333<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 112.700840<br />segmentation_metric:   6.4442897<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.938069<br />segmentation_metric:   6.7464115<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.735545<br />segmentation_metric:   7.7414248<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.628950<br />segmentation_metric:   4.1509434<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  95.223391<br />segmentation_metric:   5.4044369<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.099710<br />segmentation_metric:   7.2755102<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.693010<br />segmentation_metric:   6.6602871<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.720036<br />segmentation_metric:   5.5472264<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.666994<br />segmentation_metric:   6.5096661<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  99.476304<br />segmentation_metric:   6.1385350<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 195.397620<br />segmentation_metric:  15.8702163<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 123.265435<br />segmentation_metric:   6.5103306<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.906164<br />segmentation_metric:   7.5741176<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.726449<br />segmentation_metric:   4.6637931<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 117.530802<br />segmentation_metric:   6.5471264<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 179.675644<br />segmentation_metric:   7.3812317<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 109.333357<br />segmentation_metric:   6.7043651<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 101.156752<br />segmentation_metric:   4.4861461<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 187.928875<br />segmentation_metric:   8.4971815<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 114.753259<br />segmentation_metric:   7.3486056<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.076275<br />segmentation_metric:  12.3378378<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.064124<br />segmentation_metric:   4.7345972<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 101.132127<br />segmentation_metric:   7.5100402<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  58.332227<br />segmentation_metric:   3.3864119<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.313060<br />segmentation_metric:   7.9547872<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.287585<br />segmentation_metric:   5.3918033<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.879137<br />segmentation_metric:   7.0621242<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.175296<br />segmentation_metric:   5.4422018<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.016705<br />segmentation_metric:   4.9170732<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.749988<br />segmentation_metric:   6.8679928<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.366251<br />segmentation_metric:   7.9860000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  73.174766<br />segmentation_metric:   4.8670635<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.909592<br />segmentation_metric:   3.7139364<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.970747<br />segmentation_metric:   8.2212257<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.678825<br />segmentation_metric:   5.0592417<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:   4.742873<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.207330<br />segmentation_metric:   6.6627119<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.858008<br />segmentation_metric:   8.6053412<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  38.354349<br />segmentation_metric:   6.4789916<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.058302<br />segmentation_metric:   4.1483421<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.055384<br />segmentation_metric:  25.5431034<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  56.188233<br />segmentation_metric:  15.0526316<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 123.552260<br />segmentation_metric:   5.9704017<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:   4.530662<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.844111<br />segmentation_metric:   3.7496038<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  43.690178<br />segmentation_metric:   4.0410256<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.687254<br />segmentation_metric:   9.8053528<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  66.909891<br />segmentation_metric:   8.6309963<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  74.760998<br />segmentation_metric:   5.9975962<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.385983<br />segmentation_metric:  10.5326087<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  22.158454<br />segmentation_metric:  15.4000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.358962<br />segmentation_metric:  18.7169811<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.019586<br />segmentation_metric: 296.2142857<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  25.895395<br />segmentation_metric:   1.2385787<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.261730<br />segmentation_metric:  32.2952381<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 180.017843<br />segmentation_metric:  56.4509804<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  28.633199<br />segmentation_metric:   1.9109948<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  24.651002<br />segmentation_metric:  22.3076923<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.919729<br />segmentation_metric:  11.1208333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  58.592028<br />segmentation_metric:   5.4523810<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.164042<br />segmentation_metric:   6.6209524<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.716624<br />segmentation_metric:   7.3139159<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.614964<br />segmentation_metric:  19.5083333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 108.397980<br />segmentation_metric:   5.4810924<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 138.080056<br />segmentation_metric:  22.4713656<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  13.035671<br />segmentation_metric:   1.6521739<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  27.666820<br />segmentation_metric:  25.5000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.391750<br />segmentation_metric:   5.9951691<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 147.443366<br />segmentation_metric:  12.4618938<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.825020<br />segmentation_metric:   4.7056856<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.414137<br />segmentation_metric:   5.0291005<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  30.810563<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.544263<br />segmentation_metric:   6.8867403<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.757382<br />segmentation_metric:   4.9690949<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.647206<br />segmentation_metric:   5.5780591<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  73.612568<br />segmentation_metric:   5.8983452<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  58.955107<br />segmentation_metric:   5.0413625<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.187099<br />segmentation_metric:   7.2495088<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.701041<br />segmentation_metric:   5.4171123<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.473680<br />segmentation_metric:   5.2095588<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.914203<br />segmentation_metric:   6.1141304<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  81.269504<br />segmentation_metric:   5.7176015<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.653443<br />segmentation_metric:   5.8473581<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.839784<br />segmentation_metric:   5.0613811<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.164547<br />segmentation_metric:   6.3162879<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.433376<br />segmentation_metric:   6.0658683<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.381686<br />segmentation_metric:   6.2719449<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.864190<br />segmentation_metric:   5.5383361<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 114.187306<br />segmentation_metric:   6.9249448<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 116.145672<br />segmentation_metric:   4.3243993<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.338182<br />segmentation_metric:   5.7853261<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.267426<br />segmentation_metric:   5.5717213<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.202840<br />segmentation_metric:   4.8604651<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.765605<br />segmentation_metric:   6.5287356<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.622514<br />segmentation_metric:   8.5313351<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  44.148049<br />segmentation_metric:   1.9796380<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 157.742519<br />segmentation_metric:   6.6909414<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 125.235628<br />segmentation_metric:   5.4887737<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:   8.068961<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 111.017522<br />segmentation_metric:   4.6000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.124960<br />segmentation_metric:   6.2674157<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  76.320639<br />segmentation_metric:   6.4640657<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.678548<br />segmentation_metric:   6.6146789<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 143.349741<br />segmentation_metric:   5.9639889<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.655706<br />segmentation_metric:   4.1057884<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 104.580499<br />segmentation_metric:   5.6219931<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 166.531074<br />segmentation_metric:   7.7142857<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.169336<br />segmentation_metric:   5.2551834<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.564998<br />segmentation_metric:   5.9014423<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.660037<br />segmentation_metric:   7.5537975<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 103.915019<br />segmentation_metric:   7.0535714<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.939919<br />segmentation_metric:   3.7567568<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 108.427325<br />segmentation_metric:   6.3731707<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 124.397405<br />segmentation_metric:   5.5334190<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.480806<br />segmentation_metric:   6.2150259<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.763064<br />segmentation_metric:   5.0375817<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.471736<br />segmentation_metric:   3.6293996<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.024238<br />segmentation_metric:   5.2374517<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  56.733713<br />segmentation_metric:   5.5571429<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.073305<br />segmentation_metric:   5.9819376<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.075713<br />segmentation_metric:   7.4065934<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  54.850117<br />segmentation_metric:   5.8951049<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.915754<br />segmentation_metric:   5.5712166<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.621147<br />segmentation_metric:   5.9040767<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 152.036253<br />segmentation_metric:   5.7317784<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 122.486048<br />segmentation_metric:   4.2835366<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.278917<br />segmentation_metric:   4.6457143<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  37.598668<br />segmentation_metric:   2.8092105<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  25.109259<br />segmentation_metric:   1.4924242<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.611767<br />segmentation_metric:   6.1898527<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  71.294048<br />segmentation_metric:   3.4400000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.218477<br />segmentation_metric:   5.4968153<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.217753<br />segmentation_metric:   4.0352113<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.321194<br />segmentation_metric:   3.8333333<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  29.443897<br />segmentation_metric:   1.4436090<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.114412<br />segmentation_metric:   6.3834951<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.155403<br />segmentation_metric:   5.3876652<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.637809<br />segmentation_metric:   4.4598394<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.710230<br />segmentation_metric:   7.6020408<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  66.232138<br />segmentation_metric:   2.8589744<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 210.649824<br />segmentation_metric:  13.3832117<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 132.709911<br />segmentation_metric:   7.4467354<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.715239<br />segmentation_metric:   4.7228739<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.034989<br />segmentation_metric:   8.0653951<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.512323<br />segmentation_metric:   5.2507740<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.039338<br />segmentation_metric:   6.1799544<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  27.972019<br />segmentation_metric:   1.1457627<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  26.983931<br />segmentation_metric:   2.4727273<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.864355<br />segmentation_metric:   5.4623188<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.143788<br />segmentation_metric:   5.2003968<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.159804<br />segmentation_metric:   4.8191882<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.537519<br />segmentation_metric:   5.9310345<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 123.272796<br />segmentation_metric:   6.0409357<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.062064<br />segmentation_metric:   6.3880342<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  93.122212<br />segmentation_metric:   5.5374554<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 142.146803<br />segmentation_metric:   7.1640212<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 146.977457<br />segmentation_metric:   9.1673469<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.469424<br />segmentation_metric:   5.2781955<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.784950<br />segmentation_metric:   6.1314060<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 117.413657<br />segmentation_metric:   5.3850340<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 108.317264<br />segmentation_metric:   5.4054983<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.436809<br />segmentation_metric:   4.6247544<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 164.567364<br />segmentation_metric:   8.3567073<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 101.558582<br />segmentation_metric:   4.2318584<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.305605<br />segmentation_metric:   4.0159490<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.604851<br />segmentation_metric:   6.5982405<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  44.234808<br />segmentation_metric:   1.4848485<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.091860<br />segmentation_metric:   5.6717268<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.519385<br />segmentation_metric:   5.6098901<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.875087<br />segmentation_metric:   5.8068740<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 121.855730<br />segmentation_metric:   7.1462799<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  62.206524<br />segmentation_metric:   5.1752336<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  98.574790<br />segmentation_metric:   6.6488439<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  28.139315<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 103.368072<br />segmentation_metric:   6.4968944<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.832672<br />segmentation_metric:   5.9701195<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  28.567294<br />segmentation_metric:   1.5373134<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 151.016913<br />segmentation_metric:  11.9596413<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.041833<br />segmentation_metric:   4.8458647<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.542597<br />segmentation_metric:   7.2962963<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 128.551283<br />segmentation_metric:  11.1400709<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.854565<br />segmentation_metric:   4.9095607<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.656741<br />segmentation_metric:   6.6383648<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 194.064099<br />segmentation_metric:  12.6089172<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  47.479082<br />segmentation_metric:   2.6929638<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  57.333940<br />segmentation_metric:   4.0546448<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  41.308374<br />segmentation_metric:   3.1928375<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  59.833458<br />segmentation_metric:   3.7364130<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  52.097559<br />segmentation_metric:   3.3327172<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.799376<br />segmentation_metric:   5.1839864<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 110.294377<br />segmentation_metric:   7.1493776<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.350374<br />segmentation_metric:   7.5544794<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.739852<br />segmentation_metric:   5.8406040<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.127321<br />segmentation_metric:   4.4891697<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  49.150436<br />segmentation_metric:   4.2187500<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.534336<br />segmentation_metric:   4.8207547<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.474240<br />segmentation_metric:   7.3790524<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.279982<br />segmentation_metric:   5.2625000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 101.112371<br />segmentation_metric:   7.7589286<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.410043<br />segmentation_metric:   9.4084507<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.935807<br />segmentation_metric:   5.8944282<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  84.229375<br />segmentation_metric:   6.9447514<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  61.252888<br />segmentation_metric:   5.3688525<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.794076<br />segmentation_metric:   7.9813333<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.261559<br />segmentation_metric:   6.5665025<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.448249<br />segmentation_metric:   6.7010753<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  49.965202<br />segmentation_metric:   4.8558282<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  59.830689<br />segmentation_metric:   5.0442478<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.320949<br />segmentation_metric:   4.8440111<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.084805<br />segmentation_metric:   6.5945946<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.442201<br />segmentation_metric:   7.3333333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.002818<br />segmentation_metric:   5.4974003<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.380039<br />segmentation_metric:   6.1527495<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.279388<br />segmentation_metric:   6.8341158<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.390380<br />segmentation_metric:   4.3594378<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.288145<br />segmentation_metric:   6.5737265<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.825897<br />segmentation_metric:   6.3608018<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 105.594885<br />segmentation_metric:   7.3237082<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.388908<br />segmentation_metric:   4.0599174<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.259252<br />segmentation_metric:   5.9279279<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.447615<br />segmentation_metric:   5.1080332<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.360591<br />segmentation_metric:   5.0024450<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.915338<br />segmentation_metric:   6.8396947<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.658880<br />segmentation_metric:   5.5490196<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.716974<br />segmentation_metric:   5.9316888<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  51.405273<br />segmentation_metric:   4.0322581<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.333963<br />segmentation_metric:   5.4319809<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.197751<br />segmentation_metric:   9.3021583<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.573149<br />segmentation_metric:   5.3088803<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.029095<br />segmentation_metric:   6.1192893<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.859977<br />segmentation_metric:   5.3819444<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.311715<br />segmentation_metric:   5.6023295<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.824248<br />segmentation_metric:   6.8384798<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.744656<br />segmentation_metric:   7.4679245<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  58.884638<br />segmentation_metric:   4.9645390<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  84.890164<br />segmentation_metric:   4.1466905<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.235717<br />segmentation_metric:   3.6213922<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.325703<br />segmentation_metric:   4.4128596<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 123.592538<br />segmentation_metric:   5.3482587<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.176235<br />segmentation_metric:   4.7670251<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 107.490794<br />segmentation_metric:   5.1582278<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 111.402398<br />segmentation_metric:   7.2803598<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  85.722016<br />segmentation_metric:   5.5864909<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 107.312489<br />segmentation_metric:   4.4086162<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 108.737178<br />segmentation_metric:   8.8830846<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  58.219201<br />segmentation_metric:   4.8437500<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 217.985584<br />segmentation_metric:   6.3214286<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 182.989180<br />segmentation_metric:  17.2252874<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.609334<br />segmentation_metric:   8.6853933<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.937594<br />segmentation_metric:   6.0650000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  42.993377<br />segmentation_metric:   1.3008130<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.318012<br />segmentation_metric:   6.3671275<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 117.256910<br />segmentation_metric:   6.0465890<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.510402<br />segmentation_metric:   6.5886214<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.891359<br />segmentation_metric:   7.2511628<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.269929<br />segmentation_metric:   3.9682300<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.883094<br />segmentation_metric:   7.1636364<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  90.416361<br />segmentation_metric:   6.1713344<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  70.790685<br />segmentation_metric:   8.1092437<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   5.215362<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  73.896318<br />segmentation_metric:   6.0300752<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.865680<br />segmentation_metric:   4.4380952<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 111.014993<br />segmentation_metric:   6.6821705<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.178276<br />segmentation_metric:   5.8871473<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.871935<br />segmentation_metric:   5.3460208<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 160.413812<br />segmentation_metric:  11.9630607<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.513109<br />segmentation_metric:   5.1550000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.233496<br />segmentation_metric:   7.4591195<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.933215<br />segmentation_metric:   6.0362117<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  84.758216<br />segmentation_metric:   6.3721286<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.907899<br />segmentation_metric:   9.1005587<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.891973<br />segmentation_metric:   4.9411765<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  57.207916<br />segmentation_metric:   4.2992565<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 116.078353<br />segmentation_metric:   5.3949843<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.437097<br />segmentation_metric:  14.1551724<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   5.542593<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.473916<br />segmentation_metric:   8.7771318<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.004652<br />segmentation_metric:   5.4088586<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 132.961583<br />segmentation_metric:   7.8938849<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.753096<br />segmentation_metric:   4.6749311<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 106.941302<br />segmentation_metric:   8.4674330<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 150.043683<br />segmentation_metric:   6.9355401<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.372322<br />segmentation_metric:   6.0677570<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.311185<br />segmentation_metric:  16.9225092<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.213636<br />segmentation_metric:   5.5055821<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.716778<br />segmentation_metric:   9.0893971<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.627380<br />segmentation_metric:   7.9473684<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 116.675518<br />segmentation_metric:   5.8109339<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 119.403674<br />segmentation_metric:   5.6528384<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  37.517871<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 120.147448<br />segmentation_metric:   8.1148148<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.721456<br />segmentation_metric:   6.2398374<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.304792<br />segmentation_metric:   6.6976285<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.415615<br />segmentation_metric:   5.5634441<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 109.024277<br />segmentation_metric:   4.4720893<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  98.977840<br />segmentation_metric:   4.4009662<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  35.551712<br />segmentation_metric:   1.8225108<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  82.985519<br />segmentation_metric:   5.9153318<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  38.933753<br />segmentation_metric:   1.8294931<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 167.450741<br />segmentation_metric:  10.2264957<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  31.788619<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  82.091145<br />segmentation_metric:   8.5192802<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 122.764918<br />segmentation_metric:   7.8065099<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 126.987458<br />segmentation_metric:   6.7555866<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 112.971407<br />segmentation_metric:   4.9656020<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.435371<br />segmentation_metric:   5.5205725<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.736905<br />segmentation_metric:   6.3939394<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  59.301735<br />segmentation_metric:   3.4273859<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.236683<br />segmentation_metric:   6.9620253<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  63.905312<br />segmentation_metric:   6.2891832<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  38.928609<br />segmentation_metric:   4.6057692<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.843345<br />segmentation_metric:   4.5033113<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.623760<br />segmentation_metric:   7.0386266<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.373741<br />segmentation_metric:   8.3891213<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.081018<br />segmentation_metric:   4.7262970<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.429642<br />segmentation_metric:   6.2947977<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  61.795027<br />segmentation_metric:   5.8550725<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.273730<br />segmentation_metric:   6.0583658<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  78.091597<br />segmentation_metric:   5.9858491<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  62.228022<br />segmentation_metric:   4.8852941<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  93.303585<br />segmentation_metric:   6.7059925<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.582284<br />segmentation_metric:   5.1743929<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 116.989735<br />segmentation_metric:   6.8122271<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.986944<br />segmentation_metric:   4.8944338<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  90.458378<br />segmentation_metric:   5.5791726<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 138.089003<br />segmentation_metric:   5.5244618<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.401412<br />segmentation_metric:   6.3812010<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.798451<br />segmentation_metric:   7.2350000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  85.465743<br />segmentation_metric:   6.8881432<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.747230<br />segmentation_metric:   5.2756598<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.757614<br />segmentation_metric:   7.5338753<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.895534<br />segmentation_metric:   5.3412322<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.620092<br />segmentation_metric:   6.6016713<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.380532<br />segmentation_metric:   6.8607595<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:   4.318667<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.660778<br />segmentation_metric:   4.5750000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.995528<br />segmentation_metric:   6.1221945<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.946346<br />segmentation_metric:   9.0315789<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  87.303186<br />segmentation_metric:   5.6984993<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  90.209404<br />segmentation_metric:   9.7149028<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  95.965855<br />segmentation_metric:   5.7563636<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.714222<br />segmentation_metric:   6.4551422<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 132.319686<br />segmentation_metric:   9.8659966<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.204215<br />segmentation_metric:   7.1100703<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  58.242599<br />segmentation_metric:   5.8992443<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  46.021336<br />segmentation_metric:   4.2789855<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.333638<br />segmentation_metric:   7.1648045<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  84.719761<br />segmentation_metric:  10.3316953<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.679156<br />segmentation_metric:   6.0736342<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 105.662890<br />segmentation_metric:   3.9311927<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.966331<br />segmentation_metric:   4.4671815<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  42.914187<br />segmentation_metric:   3.2398524<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 210.535060<br />segmentation_metric:   6.4884393<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.340697<br />segmentation_metric:   7.7157107<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  85.114541<br />segmentation_metric:   9.1327231<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 147.172382<br />segmentation_metric:   7.9685950<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 133.511967<br />segmentation_metric:   4.4885845<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  79.072720<br />segmentation_metric:   8.0061728<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.303710<br />segmentation_metric:   6.0909091<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.080490<br />segmentation_metric:   7.3333333<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.860113<br />segmentation_metric:   5.6762452<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  98.075449<br />segmentation_metric:   6.2959720<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  68.792750<br />segmentation_metric:   3.4561664<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.194650<br />segmentation_metric:   5.5640138<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 128.945336<br />segmentation_metric:   8.1184211<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.626310<br />segmentation_metric:   5.8957529<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  33.069163<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  73.557633<br />segmentation_metric:   5.9256018<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  95.816902<br />segmentation_metric:   5.2331839<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.307697<br />segmentation_metric:   5.4930966<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 164.251818<br />segmentation_metric:   6.2164430<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  86.544424<br />segmentation_metric:   5.1423150<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.629246<br />segmentation_metric:   6.5991189<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 107.129745<br />segmentation_metric:   6.0574324<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.132194<br />segmentation_metric:   6.9742857<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  79.538189<br />segmentation_metric:   5.7908903<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 105.005308<br />segmentation_metric:   7.7185185<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  32.535359<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.680593<br />segmentation_metric:   6.9038462<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 146.122198<br />segmentation_metric:   3.2584906<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  84.426113<br />segmentation_metric:   6.1969697<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.573422<br />segmentation_metric:   6.2035794<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 129.575191<br />segmentation_metric:   5.1672078<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.671101<br />segmentation_metric:   5.5939279<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.288314<br />segmentation_metric:   3.4989605<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.444597<br />segmentation_metric:   4.9847973<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.807255<br />segmentation_metric:   2.8008214<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  86.459281<br />segmentation_metric:   6.1727749<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 133.269490<br />segmentation_metric:   9.1909341<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 211.906336<br />segmentation_metric:  13.0780142<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 134.734578<br />segmentation_metric:   6.5813559<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.191845<br />segmentation_metric:   5.2365039<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  58.846316<br />segmentation_metric:   5.5777778<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 158.269098<br />segmentation_metric:  18.2046784<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 108.932820<br />segmentation_metric:  15.9050633<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.752843<br />segmentation_metric:   5.9389671<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.223095<br />segmentation_metric:   6.4344569<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  17.517962<br />segmentation_metric:   0.8563536<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:   5.556314<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  58.201802<br />segmentation_metric:   6.2554945<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.185359<br />segmentation_metric:   5.3096927<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.383511<br />segmentation_metric:   5.5200000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  42.491995<br />segmentation_metric:   1.5679245<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  37.045515<br />segmentation_metric:   4.0154639<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.023535<br />segmentation_metric:   3.8132530<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  50.365249<br />segmentation_metric:   4.1684982<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 100.791965<br />segmentation_metric:   6.1723447<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 107.815851<br />segmentation_metric:   7.9132075<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 127.582925<br />segmentation_metric:   5.1518817<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  56.422631<br />segmentation_metric:   5.1971831<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  21.272986<br />segmentation_metric:   1.4731183<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  90.581419<br />segmentation_metric:   7.9727047<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.262498<br />segmentation_metric:   7.3538136<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.012523<br />segmentation_metric:   6.1074627<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.324687<br />segmentation_metric:   5.0688576<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 124.466217<br />segmentation_metric:   8.0461165<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.507311<br />segmentation_metric:   5.4597156<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.193075<br />segmentation_metric:   7.8762887<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  76.363071<br />segmentation_metric:   5.7764350<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.967179<br />segmentation_metric:   5.0106383<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.357059<br />segmentation_metric:   5.3692308<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  56.634975<br />segmentation_metric:   4.7900552<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  70.114066<br />segmentation_metric:   3.2515152<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 108.420965<br />segmentation_metric:   5.0208817<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.374501<br />segmentation_metric:   5.0615183<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.537984<br />segmentation_metric:   5.7674419<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  50.168885<br />segmentation_metric:   4.5743243<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.866732<br />segmentation_metric:   6.1421053<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  61.412251<br />segmentation_metric:   5.0897756<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.398334<br />segmentation_metric:   5.4834711<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  48.632959<br />segmentation_metric:   5.3239437<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.346996<br />segmentation_metric:   7.3092269<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.271958<br />segmentation_metric:   5.8733945<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.458461<br />segmentation_metric:   7.6338583<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  89.843999<br />segmentation_metric:   6.3478261<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 161.826606<br />segmentation_metric:  10.5788462<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 101.134908<br />segmentation_metric:   6.2857143<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  57.667377<br />segmentation_metric:   6.2705479<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.767759<br />segmentation_metric:  10.0000000<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.685979<br />segmentation_metric:   4.2314540<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  61.435992<br />segmentation_metric:   5.7539267<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 137.177886<br />segmentation_metric:   9.6416819<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 113.397066<br />segmentation_metric:   5.1862745<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.014630<br />segmentation_metric:   6.6008316<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 119.622466<br />segmentation_metric:   6.8855634<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.689753<br />segmentation_metric:   5.6099476<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.048821<br />segmentation_metric:   4.3051771<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  88.607716<br />segmentation_metric:   6.5504762<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 135.032809<br />segmentation_metric:   6.6572052<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  25.232275<br />segmentation_metric:   1.2690583<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 117.804735<br />segmentation_metric:  10.5768501<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.722388<br />segmentation_metric:   5.3727599<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 116.819251<br />segmentation_metric:   6.6661972<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.155354<br />segmentation_metric:   7.2000000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.923660<br />segmentation_metric:   6.5927342<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.345968<br />segmentation_metric:   5.2482270<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.862071<br />segmentation_metric:   3.4009901<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 101.478910<br />segmentation_metric:  11.9765808<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.589417<br />segmentation_metric:   7.3232044<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  61.362834<br />segmentation_metric:   5.3623853<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.575000<br />segmentation_metric:   6.1440000<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.999702<br />segmentation_metric:   7.2201139<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 115.409292<br />segmentation_metric:   6.6159601<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.484569<br />segmentation_metric:   5.7199017<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 129.212577<br />segmentation_metric:   7.2845850<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.672395<br />segmentation_metric:   5.6005155<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.929323<br />segmentation_metric:   7.0414815<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  69.335458<br />segmentation_metric:   6.1408115<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.154939<br />segmentation_metric:   4.2347418<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 127.435587<br />segmentation_metric:   7.9553451<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  94.709445<br />segmentation_metric:   5.6621005<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  71.050232<br />segmentation_metric:   6.1306210<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  49.496058<br />segmentation_metric:   5.0218750<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  55.348347<br />segmentation_metric:   5.0031348<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  99.670964<br />segmentation_metric:   5.1391586<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 151.917567<br />segmentation_metric:   8.1843220<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.464519<br />segmentation_metric:   4.8379052<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.451962<br />segmentation_metric:   9.0479798<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.557178<br />segmentation_metric:   4.3805970<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.794203<br />segmentation_metric:   7.4148607<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  92.691890<br />segmentation_metric:   4.9232456<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.903722<br />segmentation_metric:   8.4898876<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  55.497125<br />segmentation_metric:   3.7787356<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.389004<br />segmentation_metric:   5.0407609<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.285135<br />segmentation_metric:   6.5326633<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.580179<br />segmentation_metric:   5.9566667<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.563536<br />segmentation_metric:   6.3073770<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.583576<br />segmentation_metric:   5.9337349<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.911042<br />segmentation_metric:   6.6265060<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.038202<br />segmentation_metric:   8.6033058<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.377229<br />segmentation_metric:  10.6984733<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 113.088100<br />segmentation_metric:   6.7725225<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  36.724176<br />segmentation_metric:   1.5788114<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  12.081709<br />segmentation_metric:   2.2000000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  75.979598<br />segmentation_metric:   5.5015106<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 119.765226<br />segmentation_metric:   6.6625954<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  53.592651<br />segmentation_metric:   3.5687500<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.538036<br />segmentation_metric:   9.5945946<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  52.474916<br />segmentation_metric:   3.4349030<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  54.928173<br />segmentation_metric:   3.7243590<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 115.073074<br />segmentation_metric:   6.3077790<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 107.527555<br />segmentation_metric:   4.9591078<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  92.339767<br />segmentation_metric:   6.0865140<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  94.645559<br />segmentation_metric:   7.0664962<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.779844<br />segmentation_metric:   5.8425787<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.071507<br />segmentation_metric:   5.8988942<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.780878<br />segmentation_metric:   6.7080103<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.538108<br />segmentation_metric:   5.7156134<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  94.642218<br />segmentation_metric:   6.5415282<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.940414<br />segmentation_metric:   4.5570175<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.204652<br />segmentation_metric:   6.9281314<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  94.277239<br />segmentation_metric:   7.8838710<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  72.296552<br />segmentation_metric:   6.5452196<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 148.472819<br />segmentation_metric:   6.6371553<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.821727<br />segmentation_metric:   7.4471545<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 149.656040<br />segmentation_metric:   8.2253829<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 127.017437<br />segmentation_metric:   9.9982394<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.743531<br />segmentation_metric:   5.4555766<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.154183<br />segmentation_metric:   4.9703947<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  90.186227<br />segmentation_metric:   4.8169811<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 153.897179<br />segmentation_metric:   9.9913295<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  72.481395<br />segmentation_metric:   8.4036939<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 106.007178<br />segmentation_metric:   8.2445521<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.028258<br />segmentation_metric:   6.3057090<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  79.882388<br />segmentation_metric:   6.1199226<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  26.024659<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.372898<br />segmentation_metric:   4.6206294<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.865481<br />segmentation_metric:   4.4420168<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.088578<br />segmentation_metric:   7.3119777<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.654611<br />segmentation_metric:   7.1163522<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 110.225765<br />segmentation_metric:   4.0855263<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 102.124847<br />segmentation_metric:   7.3523238<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.618858<br />segmentation_metric:   9.4294355<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.470020<br />segmentation_metric:   6.4549296<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.974850<br />segmentation_metric:   6.4819820<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  75.344839<br />segmentation_metric:   5.6487179<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 115.317770<br />segmentation_metric:   5.9947644<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.768933<br />segmentation_metric:   5.8441331<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.902391<br />segmentation_metric:   6.8209607<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.708079<br />segmentation_metric:   3.7869482<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 106.295088<br />segmentation_metric:   6.9305913<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 155.835086<br />segmentation_metric:   7.0526316<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  50.967360<br />segmentation_metric:   4.5333333<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 118.994453<br />segmentation_metric:   6.2550861<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 120.243263<br />segmentation_metric:   6.6484848<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  47.477060<br />segmentation_metric:   4.0945455<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.211610<br />segmentation_metric:   5.8671454<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  65.838881<br />segmentation_metric:   5.7933673<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  89.822430<br />segmentation_metric:   6.9002217<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  69.864035<br />segmentation_metric:   5.1458824<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 102.937734<br />segmentation_metric:   9.4752066<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 129.824585<br />segmentation_metric:   7.2548263<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.487424<br />segmentation_metric:   6.9324818<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:   5.540493<br />segmentation_metric:   1.0000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.946041<br />segmentation_metric:   5.6266234<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.829460<br />segmentation_metric:   6.5190563<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.402180<br />segmentation_metric:   5.5000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  80.207395<br />segmentation_metric:   6.6477987<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  67.061872<br />segmentation_metric:   5.5284360<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 119.850546<br />segmentation_metric:   6.9160305<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 152.535029<br />segmentation_metric:   7.3473054<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 132.083525<br />segmentation_metric:   3.9847856<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.861179<br />segmentation_metric:   5.2828784<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.264830<br />segmentation_metric:   6.0663265<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 102.095853<br />segmentation_metric:   4.9950980<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  60.544005<br />segmentation_metric:   5.6781327<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.130445<br />segmentation_metric:   6.8881789<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 127.570830<br />segmentation_metric:   7.4175655<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.525089<br />segmentation_metric:   7.4906054<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.256804<br />segmentation_metric:   7.2865014<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.028383<br />segmentation_metric:   9.4376200<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 118.594594<br />segmentation_metric:   8.3776908<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.163626<br />segmentation_metric:   9.5869018<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.732075<br />segmentation_metric:   7.1437372<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  91.275506<br />segmentation_metric:   7.7348928<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 110.281094<br />segmentation_metric:   6.5000000<br />well_name: C09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  36.489905<br />segmentation_metric:   3.0379310<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 145.449048<br />segmentation_metric:   5.6763285<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.688412<br />segmentation_metric:   4.1457490<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.970756<br />segmentation_metric:   7.0072727<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.669006<br />segmentation_metric:   4.4499366<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.229083<br />segmentation_metric:   4.0949721<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 143.017648<br />segmentation_metric:   4.6024845<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  98.111308<br />segmentation_metric:   4.1451613<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  48.245515<br />segmentation_metric:   5.4448399<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  31.878379<br />segmentation_metric:   2.2575758<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 204.404786<br />segmentation_metric:   5.3819188<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.985535<br />segmentation_metric:   5.8315972<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  65.444762<br />segmentation_metric:   4.4218750<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  60.738209<br />segmentation_metric:   5.3486683<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 158.771314<br />segmentation_metric:   5.2228464<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.443446<br />segmentation_metric:   5.8036117<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 113.100473<br />segmentation_metric:   5.1456456<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 128.211327<br />segmentation_metric:   6.4980237<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.227764<br />segmentation_metric:   5.4142857<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.051071<br />segmentation_metric:   5.0451128<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  80.071532<br />segmentation_metric:   4.5962441<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  12.465912<br />segmentation_metric:   1.4318182<br />well_name: C09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  92.538805<br />segmentation_metric:   5.1004942<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  65.905033<br />segmentation_metric:   6.7994505<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  56.069304<br />segmentation_metric:   4.5208333<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.538681<br />segmentation_metric:   5.6012270<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.472647<br />segmentation_metric:   6.1959459<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.483832<br />segmentation_metric:   5.4059041<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 104.593991<br />segmentation_metric:   6.6210235<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 180.899733<br />segmentation_metric:   6.5179283<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.062519<br />segmentation_metric:   5.9445629<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.918305<br />segmentation_metric:   5.7008368<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  65.445748<br />segmentation_metric:   6.6443149<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  98.520738<br />segmentation_metric:   7.7468354<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.092971<br />segmentation_metric:   6.3079585<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.821613<br />segmentation_metric:   4.2881002<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 121.270596<br />segmentation_metric:   7.2089947<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.575215<br />segmentation_metric:   6.6071429<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.062166<br />segmentation_metric:   5.6907216<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.220207<br />segmentation_metric:   5.4868106<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  84.277978<br />segmentation_metric:   5.6207627<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.323582<br />segmentation_metric:   8.2488263<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.664399<br />segmentation_metric:   5.8155556<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.902759<br />segmentation_metric:   5.4781145<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 103.203451<br />segmentation_metric:   6.4780316<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 102.474278<br />segmentation_metric:   7.1033479<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 131.764759<br />segmentation_metric:   7.5184382<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 109.965704<br />segmentation_metric:   7.1176471<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.796963<br />segmentation_metric:   4.3350877<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 102.094692<br />segmentation_metric:   6.0661376<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.557548<br />segmentation_metric:   6.4400000<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.541972<br />segmentation_metric:   6.1023810<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  79.344710<br />segmentation_metric:   6.7598566<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.411715<br />segmentation_metric:   4.2839806<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  67.470712<br />segmentation_metric:   6.7705882<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.867388<br />segmentation_metric:   4.6450704<br />well_name: C09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.055527<br />segmentation_metric:   4.3789474<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.154649<br />segmentation_metric:   5.6868009<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.943720<br />segmentation_metric:   5.0414201<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 103.390653<br />segmentation_metric:   7.8652778<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.911163<br />segmentation_metric:   3.5793358<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.779181<br />segmentation_metric:   4.6129032<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  66.121599<br />segmentation_metric:   5.2516340<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 101.934405<br />segmentation_metric:   4.6662354<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  45.547961<br />segmentation_metric:   3.7123746<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.332915<br />segmentation_metric:   5.6734177<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.295672<br />segmentation_metric:   6.5125628<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  49.066570<br />segmentation_metric:   4.0523256<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  67.660041<br />segmentation_metric:   6.8377778<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.442415<br />segmentation_metric:   6.6694491<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  47.916294<br />segmentation_metric:   3.3790524<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.726467<br />segmentation_metric:   6.6180124<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 216.276556<br />segmentation_metric:  11.8254157<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  59.310430<br />segmentation_metric:   6.3139842<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 133.940995<br />segmentation_metric:   8.4613779<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  95.711036<br />segmentation_metric:   7.0876897<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 120.706516<br />segmentation_metric:   8.2795031<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.281184<br />segmentation_metric:   4.2941176<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  51.005668<br />segmentation_metric:   4.1707317<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 125.189341<br />segmentation_metric:   5.8585859<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.235532<br />segmentation_metric:   4.8649706<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.246660<br />segmentation_metric:   4.6732283<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 104.783720<br />segmentation_metric:   6.6263736<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.048191<br />segmentation_metric:   6.6994048<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  91.712441<br />segmentation_metric:   6.4751958<br />well_name: C09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.238884<br />segmentation_metric:   7.3202479<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 102.043410<br />segmentation_metric:  10.2460850<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.122255<br />segmentation_metric:   7.1883408<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 110.903565<br />segmentation_metric:   3.7719298<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  90.191230<br />segmentation_metric:   6.6188341<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.912828<br />segmentation_metric:   9.2286325<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 126.168848<br />segmentation_metric:   5.6737864<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 125.053965<br />segmentation_metric:   8.7412281<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 176.805903<br />segmentation_metric:   9.0469799<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 104.777819<br />segmentation_metric:   6.3607477<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.762619<br />segmentation_metric:   7.3981763<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.799878<br />segmentation_metric:   7.4666667<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  68.364236<br />segmentation_metric:   4.5604607<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 203.195095<br />segmentation_metric:   6.6542185<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 112.469796<br />segmentation_metric:   6.0340502<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  92.367706<br />segmentation_metric:   5.5442043<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  31.495477<br />segmentation_metric:   2.6323529<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.495119<br />segmentation_metric:   7.9227202<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.100593<br />segmentation_metric:   7.6801347<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  33.783153<br />segmentation_metric:   3.4906832<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  50.930818<br />segmentation_metric:   4.0157068<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 121.529054<br />segmentation_metric:   7.5693431<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  80.599487<br />segmentation_metric:   6.3416667<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 110.954928<br />segmentation_metric:   8.4973958<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  80.474684<br />segmentation_metric:   5.6235864<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.308028<br />segmentation_metric:   7.3037975<br />well_name: C09 <br>well_pos_x: 4 <br>well_pos_y: 6"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-6.93964625,238.32691925],"y":[1.1,1.1],"text":"yintercept: 1.1","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-6.93964625,238.32691925],"y":[13,13],"text":"yintercept: 13","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":43.1050228310502},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-6.93964625,238.32691925],"tickmode":"array","ticktext":["0","50","100","150","200"],"tickvals":[0,50,100,150,200],"categoryorder":"array","categoryarray":["0","50","100","150","200"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"Mean_Morphology_Major_Axis_Length","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-13.9115430149961,310.982182320442],"tickmode":"array","ticktext":["0","100","200","300"],"tickvals":[0,100,200,300],"categoryorder":"array","categoryarray":["0","100","200","300"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"segmentation_metric","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"190819e236bf":{"x":{},"y":{},"text":{},"type":"scatter"},"190865ed5544":{"yintercept":{}},"1908109a57f1":{"yintercept":{}}},"cur_data":"190819e236bf","visdat":{"190819e236bf":["function (y) ","x"],"190865ed5544":["function (y) ","x"],"1908109a57f1":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-819bb39a72f900d4387a">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>