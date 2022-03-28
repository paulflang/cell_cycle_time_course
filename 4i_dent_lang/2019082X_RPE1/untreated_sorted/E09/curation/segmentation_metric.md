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
<div id="htmlwidget-a9d5832e95e9b195fb48" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-a9d5832e95e9b195fb48">{"x":{"data":[{"x":[62.6629,75.708204,68.875384,76.218173,55.303985,30.521171,71.244785,80.697714,108.996737,122.700155,23.311408,97.364557,57.289245,33.815541,82.85903,27.988386,169.902203,75.173894,119.146322,96.285102,62.36059,169.526705,91.699408,15.439212,88.432224,22.636288,95.275309,107.364133,2.309401,71.920321,91.792263,60.990996,97.859763,61.978183,72.34451,71.189497,95.100462,54.255068,29.844529,27.996468,55.292222,73.702156,91.444316,73.057436,34.584032,30.112846,73.876753,28.111274,29.869197,61.401122,27.496569,23.405226,23.314849,16.548924,25.878205,39.050925,24.426133,24.536145,26.614943,70.101859,29.795863,33.463222,28.072486,27.083788,30.941783,72.185281,25.757023,36.534861,26.587318,77.296704,38.092072,26.639857,19.729055,33.819889,68.780521,91.462099,56.147388,36.792018,34.255558,59.646641,58.13091,26.344104,36.269451,74.598661,29.815105,57.953561,36.184247,27.565919,61.153344,101.433352,59.972146,71.668826,118.652295,86.559026,57.904766,110.841383,125.029688,65.782398,81.12933,32.825814,70.572984,27.330603,26.994101,54.828491,31.807629,99.526327,65.283531,63.063841,43.591925,67.369183,50.364284,91.791849,48.683255,65.371986,47.260351,98.274166,67.381067,95.47514,55.505103,61.652479,63.126535,91.794133,94.67614,68.703276,72.009861,100.57443,102.536661,70.111697,125.537143,90.190308,31.423872,112.899115,82.65652,71.809717,90.423707,59.957266,104.992938,113.967162,72.247847,24.406872,132.930909,30.355039,24.809146,22.088635,31.471173,83.076605,84.694085,185.772187,27.587603,82.271078,30.596542,25.492971,29.704939,34.813318,64.15739,88.807049,94.743545,101.981293,117.760508,34.477221,64.319211,64.379865,124.471381,81.542805,46.966203,100.001213,98.413083,91.03966,36.039327,48.725068,67.929006,71.071049,62.295392,61.463272,85.457993,86.305413,78.441017,33.252491,58.721222,59.684966,34.746945,26.598835,15.754464,16.082283,104.771192,89.779325,99.981156,78.264871,63.83647,77.741252,95.698352,77.173415,82.436239,65.413969,71.443521,37.401308,89.889954,75.68439,102.500444,107.20844,69.555939,78.718901,60.778182,67.970973,16.732423,92.393635,95.068914,108.910051,72.163384,71.623086,81.022072,122.44853,92.646089,78.050279,80.716372,106.129786,85.445496,101.427002,86.796031,69.914843,103.265644,23.615859,35.467223,28.611775,28.145254,26.363024,28.042084,34.105512,29.184467,26.786425,16.522347,8.958557,23.049968,28.746552,33.637242,29.673398,23.675591,32.373783,35.900327,31.314837,31.339307,30.320031,30.126821,23.133991,49.171187,25.431641,26.274911,29.2655,28.889665,22.667679,31.954662,82.65212,30.343989,29.829067,106.868485,81.372424,62.931845,130.846979,53.604163,30.554512,67.191031,63.673007,77.025839,26.917559,27.338737,20.163759,33.777969,27.513532,28.824011,38.475169,37.524665,33.542113,35.78859,65.439738,31.59644,32.715587,32.40212,34.872537,202.792183,26.450647,146.902952,48.621907,25.604807,83.061804,22.845915,31.766311,23.245104,26.250854,68.883426,77.376277,97.729367,87.691414,99.369656,100.415037,89.305923,119.32252,80.691152,91.267647,87.022833,79.193293,80.665574,36.800288,100.688059,109.672053,102.382844,79.351944,72.567432,90.543573,56.951448,32.256891,85.60072,141.77825,117.442554,93.482375,62.580865,52.746119,100.023475,92.819514,30.804028,68.979252,163.086782,94.932901,135.221914,63.333221,99.648037,110.5267,108.036401,31.4855,79.373892,72.290501,64.586002,128.551294,69.039554,69.365175,67.974109,86.277234,77.823339,93.314593,84.847611,56.653155,52.362605,97.032904,87.511416,73.796934,48.995622,86.195192,149.096372,113.113653,100.91968,72.297555,84.143612,79.277815,24.256879,31.626449,77.51194,26.477244,92.790856,109.774822,114.314109,77.049329,26.677659,76.845219,101.063865,71.763786,73.838978,67.390842,96.045591,72.040785,134.392896,64.440829,61.731134,92.260593,65.6867,141.220193,161.484434,107.26479,50.408217,90.743506,65.34242,68.009027,46.292882,94.433784,95.309021,115.368734,55.610943,36.72609,68.687921,89.927741,109.824791,53.050581,24.734042,83.06456,79.386823,117.033714,61.505781,62.840897,96.323053,43.889012,72.2456,66.805929,41.712536,85.104795,82.657414,115.522753,65.345724,92.207632,66.875321,108.358606,125.818564,119.69728,81.744232,298.35212,77.445285,103.336784,60.433294,119.29723,63.51789,104.301922,98.456525,83.899916,61.757056,64.671905,31.490607,108.733937,83.701636,68.93804,68.088392,80.70441,102.299041,65.537207,83.028359,29.760363,33.97182,80.991565,110.626617,91.894001,22.198054,76.979478,59.949698,64.577992,135.654564,78.432704,63.882599,71.043184,94.911098,54.096871,70.043938,100.57194,21.211924,4.38178,33.749418,101.129534,83.059557,96.088173,104.814363,84.949948,95.329411,69.547077,103.788165,89.932107,60.103907,134.783439,110.838568,202.121642,73.24007,71.368943,61.541264,67.871332,99.845595,53.294112,92.485498,75.673303,78.409987,60.455513,63.19656,65.628944,67.548138,112.458991,56.357687,25.857714,97.863873,60.472351,54.178893,28.301326,120.182126,86.847405,105.451402,122.503659,124.929742,59.091274,49.972489,91.190181,50.240308,104.147337,105.471486,94.566481,96.222662,100.842561,67.364664,70.154092,86.096874,79.484944,32.379754,95.311531,45.855865,186.090782,96.562386,80.59011,56.998027,133.566216,76.295719,82.935794,76.999509,92.57081,58.641476,77.398933,116.552201,57.212007,164.659819,88.769166,85.832027,94.592093,77.119928,87.173096,66.339006,22.728097,58.10912,77.874502,97.089949,81.48331,92.595526,88.931624,8.775421,109.516451,72.921606,100.900621,102.298325,69.337619,49.764184,68.585696,74.131861,9.459173,51.965006,72.297854,83.383975,89.366514,108.048627,106.077847,95.361715,81.275331,92.226853,79.52503,81.44361,56.261012,61.73522,88.491707,66.006661,84.66963,76.115322,76.727569,36.089035,138.043802,92.221204,83.845516,78.460621,93.024347,65.345246,79.364289,63.369917,75.63063,79.180836,87.259084,38.152751,120.748549,107.781535,79.522868,119.839093,110.380646,118.407122,115.224589,103.072968,62.601796,63.509139,97.813583,80.595615,60.454356,76.454302,72.665761,79.60192,88.78809,68.868246,81.373413,101.711733,87.30218,121.475917,71.773674,88.881217,98.870728,66.591027,66.452034,62.018884,15.704789,231.725464,85.956175,60.347201,142.950117,77.932114,40.777838,17.537149,120.959258,96.393618,86.20119,13.443028,87.299962,77.520765,27.447777,23.93548,85.724215,75.264064,77.554523,82.727228,80.312804,97.27704,106.750523,70.185573,34.5243,71.275047,103.010372,91.499145,69.886332,92.684707,119.965626,155.68992,52.660252,93.649229,64.682817,55.832597,102.007689,69.114027,80.980905,51.358027,46.089095,103.575365,144.143772,13.929185,106.376194,73.540315,72.879657,62.920053,70.459707,80.187486,58.096059,83.574019,85.42271,46.115741,89.752227,51.828123,75.404029,47.771933,60.043044,87.299344,115.747868,95.955867,65.51176,112.677759,113.645138,92.243699,39.035929,81.34894,28.653026,44.966892,94.449491,120.510941,63.234133,32.823624,99.502924,81.202042,45.54055,121.582442,66.686746,66.262697,89.683526,62.205325,62.700125,81.061094,95.836039,74.111092,91.338506,188.034576,79.612456,15.945265,99.379078,86.333623,76.824524,58.060036,83.107794,110.331552,72.787806,72.024583,79.056164,96.503545,62.258274,67.793116,65.498695,89.245919,35.442246,68.981981,84.942279,67.282884,70.002919,118.656636,127.843886,91.78148,88.028668,64.036073,79.163562,179.40337,60.45031,60.594192,65.064126,104.644991,67.830975,52.152452,74.18784,67.628897,75.575426,80.839672,70.270128,120.756487,96.520025,97.37586,69.818911,77.165726,106.116549,69.970682,72.556386,44.579475,84.519468,25.76028,80.357637,69.344001,99.589493,96.546891,88.496143,84.13712,136.498913,72.843063,78.861324,80.92065,64.13512,98.713106,65.893713,75.844624,83.628182,25.119602,98.810441,77.704258,141.884429,154.390886,32.736416,82.153601,93.766245,110.579095,65.363659,100.50098,127.679877,28.293437,73.709673,91.005574,29.435155,81.108042,36.899184,98.776876,82.661747,73.421574,77.342994,75.119151,106.553581,103.368807,114.947865,84.605361,66.181129,96.097665,30.876857,32.809796,93.650574,98.812871,79.672991,67.00909,88.244353,155.883144,99.986826,113.451957,89.955324,81.474474,79.368974,111.893934,180.548804,96.235213,59.511055,115.993889,63.984682,41.764933,88.615386,80.2283,71.673582,120.211368,100.311629,130.158557,60.306605,83.364847,78.12923,42.800388,93.332775,41.983345,88.907005,91.092734,75.400076,95.952621,77.54771,90.568403,100.540759,95.375966,133.238551,87.009451,126.222889,65.439493,81.070311,47.242249,82.818613,95.754018,83.777873,121.075135,98.437506,44.433152,90.025421,66.250099,72.666941,61.844945,103.905949,108.627725,64.202053,77.025557,64.314308,68.988349,60.702948,106.90153,61.229165,73.864427,67.098712,69.132641,70.24293,15.672786,109.947767,56.524067,66.195652,70.630223,103.250758,48.93352,203.809233,38.317551,39.599707,78.140172,21.861176,29.092412,68.024218,68.880173,140.350598,106.498528,66.882626,96.527659,120.544718,100.047159,121.704756,80.785455,99.617623,97.208979,82.908359,54.121313,103.826487,107.763659,117.240967,9.266493,8.039188,57.755163,30.890308,86.526127,83.311917,77.783616,92.134579,52.334004,86.44832,100.078621,67.257035,97.076684,112.721355,79.141522,74.645478,90.53554,74.423011,87.808223,101.010005,10.233944,98.926989,61.569824,85.299853,83.191485,66.881884,85.109406,277.628708,94.865374,101.8467,78.774785,85.626906,110.431614,87.772171,81.393324,85.111714,113.604464,145.153827,83.132126,79.20958,86.860358,111.038349,97.169625,88.926821,70.650853,97.228736,100.62409,76.326109,81.111053,81.200313,77.903885,81.265303,96.812948,64.153905,53.8531,76.270704,190.630321,49.142209,85.825352,84.68505,85.308107,81.428888,57.426742,75.105337,115.91674,135.521055,104.749321,87.898929,72.06061,67.425433,108.284106,74.058133,62.861864,66.529325,69.108772,99.700821,35.731573,43.965337,76.337966,99.530069,76.522775,56.303712,71.480757,114.161194,89.268371,78.30682,76.459744,10.650048,63.530887,107.185195,63.331581,81.572219,66.511614,81.587282,101.673788,106.236345,69.018043,96.803191,89.230961,94.188716,63.571118,55.168534,17.766977,133.811403,58.910017,48.94371,83.572497,89.821256,64.775458,75.308434,63.04781,93.300822,21.106869,61.499267,78.045771,99.435225,107.000731,98.476547,62.77305,86.365631,107.565618,78.494438,117.75001,102.999818,82.679937,105.037211,74.548949,78.652278,97.847148,100.605655,40.666192,84.31125,48.866134,20.0299,86.991286,17.310185],"y":[6.65873015873016,9.21082621082621,5.10436893203883,5.04244031830239,4.72085889570552,1,3.63247863247863,6.00546448087432,5.28423236514523,16.7037037037037,1,5.79227941176471,3.04634581105169,1,4.53086419753086,1,6.73913043478261,5.68903436988543,26.2403846153846,5.82869692532943,4.55185185185185,29.1026455026455,2.07417582417582,3.41176470588235,6.78115501519757,1.84269662921348,3.82797427652733,10.6063829787234,0.3,5.00998003992016,5.1326164874552,3.04188481675393,6.19591836734694,6.1005291005291,4.77137546468401,6.78947368421053,5.24670433145009,1.29288025889968,1,1,3.64989517819706,3.84510869565217,5.34939759036145,3.65762004175365,1,1,4.26792452830189,1,1,5.60763888888889,1,1,1,1,1,1,1,1,1,4.72413793103448,1,1,1,1,1,3.36322869955157,1,1,1,4.83185840707965,1,1,1,1,4.35871404399323,3.68531468531469,3.87254901960784,1,1,5.52409638554217,3.15104166666667,1,1,5.7450495049505,1,4.58312020460358,1,1,4.91253644314869,6.96526946107784,5.10955056179775,8.07105263157895,2.97769516728625,5.22494432071269,1.55185909980431,6.07821229050279,6.59459459459459,5.97858672376874,4.7682119205298,1,6.98117154811716,1,1,5.44152046783626,1,9.79468242245199,6.08552631578947,6.28225806451613,4.13114754098361,4.55172413793103,3.78491620111732,8.04682274247492,1.52160493827161,5.89506172839506,5.33090909090909,10.0483870967742,6.05120481927711,6.7683615819209,4.20580474934037,4.5811209439528,4.54210526315789,5.96567862714509,4.12041884816754,4.56711915535445,4.5199203187251,7.21507760532151,9.16458333333333,4.65463917525773,5.59452411994785,8.55060728744939,1.00202839756592,6.26656151419558,5.53271028037383,4.68988764044944,73.0909090909091,5.66009852216749,7.18989898989899,6.84307178631052,5.11134020618557,1,15.7291066282421,1,1,1,1,7.05851063829787,6.63798701298701,9.09628378378378,1,5.55061728395062,1,1,1,1,5.67587939698493,5.5720823798627,7.36079077429984,4.87234042553191,6.7929203539823,1,4.96543778801843,3.66730038022814,3.60452961672474,6.81860465116279,3.53584905660377,5.46382189239332,4.23173803526448,5.21149425287356,2.17621145374449,3.52151898734177,5.6570796460177,6.89403973509934,5.5609243697479,3.79007633587786,6.65322580645161,5.52851711026616,6.42292490118577,1,4.1605504587156,6.66379310344828,1,1,1,1,6.08525754884547,4.37364341085271,7.11935483870968,6.18481012658228,3.98237885462555,7.21758241758242,9.6019801980198,6.76712328767123,6.50714285714286,6.19148936170213,5.49193548387097,1,5.76404494382022,7.97520661157025,5.07438016528926,9.05390835579515,5.51648351648352,7.0974358974359,6.64850136239782,5.40547945205479,1,7.90709046454768,7.13785046728972,6.20460358056266,5.31021897810219,5,5.41952506596306,3.33670520231214,5.19620253164557,6.7037037037037,5.88823529411765,5.78774289985052,4.11507936507936,7.6969696969697,5.39209225700165,4.69945355191257,4.56957928802589,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,3.98230088495575,1,1,1,1,1,1,14.1545064377682,1,1,9.61479591836735,4.98139534883721,3.74774774774775,4.59944367176634,4.92125984251969,1,7.09717868338558,4.17864923747277,1,1,1,1,1,1,1,1,1,1,1,5.06489675516224,1,1,1,1,5.62974683544304,1,8.71428571428571,1.42715231788079,1,7.59058823529412,1,1,1,1,7.49633251833741,6.86829268292683,4.61698113207547,6.6140350877193,8.46666666666667,10.404347826087,6.015,4.81196581196581,7.24950099800399,7.06926406926407,5.43657331136738,7.22395833333333,6.85912240184757,2.12737127371274,7.06085526315789,7.10539845758355,8.32251521298175,3.3307240704501,5.30366492146597,3.57837837837838,4.25838264299803,7.4375,6.59685863874346,8.10714285714286,4.0992,5.33021077283372,2.94285714285714,3.61659192825112,4.26932084309133,4.90119250425894,1,5.72810218978102,7.27495621716287,4.24955436720143,6.43405676126878,6.64305949008499,6.67964601769912,7.7977207977208,7.12368421052632,1,6.54609929078014,7.38421052631579,4.2875605815832,5.8036253776435,8.20527859237537,6.11395348837209,5.70684931506849,4.79699248120301,5.63223140495868,4.97258064516129,5.91958762886598,4.14358974358974,7.18545454545455,4.70352564102564,6.62429378531073,5.30851063829787,3.858934169279,4.71314741035857,8.32996632996633,6.20623501199041,6.65277777777778,3.6,5.09256449165402,5.62698412698413,1,1,5.80082987551867,1.19480519480519,5.02640845070423,4.52006688963211,6.26067415730337,3.4136546184739,1,6.28406466512702,6.01720430107527,5.97866666666667,5.18559556786704,5.62655601659751,3.93558282208589,5.43864229765013,6.20101351351351,3.29775280898876,3.19526627218935,5.43489583333333,2.44288577154309,8.0278232405892,4.7279322853688,6.90060851926978,3.25196850393701,6.75955414012739,3.9168765743073,5.46706586826347,2.38208955223881,4.42034548944338,6.33506044905009,5.73310225303293,4.08920187793427,2.32471264367816,4.83737024221453,5.11532385466035,5.93986636971047,5.07954545454545,3.15107913669065,6.57289002557545,5.07543103448276,10.9143356643357,5.40760869565217,6.06965174129353,9.21724137931034,3.91277258566978,4.68840579710145,5.91711229946524,2.8,5.48792270531401,5.31412103746398,5.61496350364964,3.9349593495935,6.09090909090909,5.57675438596491,6.52057613168724,8.56286266924565,5.59313725490196,6.2375366568915,7.0248500428449,8.56140350877193,6.92393736017897,6.44444444444444,6.92961876832845,5.50418410041841,5.03953871499176,6.59170305676856,4.93798449612403,6.759375,4.93406593406593,1.888,4.60416666666667,6.15367965367965,5.92787524366472,4.44974874371859,5.24873096446701,7.39852398523985,6.05035971223022,6.28405797101449,10.8947368421053,1.53333333333333,8.03132530120482,7.85057471264368,5.31805929919137,1,6.26285714285714,5.68580060422961,4.73446327683616,4.80465949820789,4.69146608315098,4.97446808510638,7.31192660550459,5.49591280653951,5.71176470588235,4.90393013100437,6.39264705882353,1.98611111111111,1,4.86842105263158,5.85490196078431,6.36176470588235,5.98260869565217,6.64327485380117,8.41955835962145,6.14744525547445,5.31808731808732,9.28947368421053,7.50785340314136,7.42348754448399,6.39539748953975,6.55333333333333,11.9211045364892,6.14766839378238,8.17794486215539,4.61944444444444,6.23426573426573,6.328125,5.56065573770492,4.42013129102845,5.78356164383562,5.93706293706294,6.86430678466077,4.94363256784969,7.89516129032258,3.89311163895487,7.29248366013072,5.17715617715618,1,9.46910112359551,7.55704697986577,5.11232876712329,2.17361111111111,7.8733681462141,5.51592356687898,3.54583333333333,4.98103666245259,6.78676470588235,2.291015625,5.15745856353591,4.76912568306011,32.9642857142857,6.79329608938547,6.40825688073395,5.34991974317817,5.14905660377358,4.58832335329341,4.31719532554257,6.57766990291262,4.29923273657289,3.83666666666667,2.46274509803922,5.46384039900249,3.19209039548023,7.75516693163752,5.78859857482185,5.84222222222222,3.84164859002169,4.85683760683761,11.4262295081967,7.71681415929203,6.88502673796791,5.94331983805668,5.20108695652174,6.09905660377358,6.02127659574468,4.90439276485788,5.09259259259259,8.40889830508475,7.32065217391304,7.46055979643766,4.86785009861933,6.51101321585903,3.93115942028985,2.68141592920354,5.70181818181818,5.32233502538071,7.10504201680672,6.98360655737705,9.42045454545454,21.4111111111111,1,8.25910064239829,4.35338345864662,9.08278867102396,6.04409448818898,5.74518201284797,4.41176470588235,4.88863636363636,5.69172932330827,2.3,4.90849673202614,6.95488721804511,6.4430823117338,9.14767932489451,6.58992805755396,6.87350427350427,7.13609467455621,7.32745591939547,7.65582655826558,7.67584745762712,7.6497975708502,5.69642857142857,6.26139817629179,6.14133333333333,5.21173469387755,6.56371490280778,7.42227378190255,4.63285024154589,1.10900473933649,6.13061224489796,5.97565543071161,8.04580152671756,5.26859504132231,4.48060941828255,5.42528735632184,7.39821029082774,5.73650107991361,6.79862700228833,4.6676217765043,7.06986899563319,5.23846153846154,6.05691056910569,5.46274509803922,4.39285714285714,4.39593908629442,6.42344497607655,3.55753040224509,4.4251968503937,5.53360768175583,5.60471204188482,9.65727699530516,6.36933045356372,6.66105769230769,6.14153846153846,5.79527559055118,5.34538152610442,5.44018058690745,5.10986547085202,7.82051282051282,5.86338797814208,5.55950920245399,6.13449023861171,5.29245283018868,6.94561933534743,4.073756432247,6.58474576271186,6.31632653061224,5.10547667342799,5.05426356589147,1,186.758064516129,7.17587939698493,3.06301050175029,4.96908809891808,7.23234624145786,4.21487603305785,4.24324324324324,5.51446945337621,5.77995642701525,7.87628865979381,1.13636363636364,5.7125,7.98687664041995,1.58796296296296,1.79024390243902,4.69521044992743,6.23157894736842,3.8466819221968,8.13151927437642,5.20861678004535,5.18432203389831,5.49748743718593,5.82968369829684,2.26071428571429,5.60671936758893,4.66519174041298,7.2316715542522,7.41755319148936,5.50212765957447,6.63808322824716,6.95774647887324,3.18694362017804,4.94205298013245,4.48681055155875,5.66233766233766,10.3224852071006,4.35875706214689,4.72727272727273,1.88985507246377,1.90864197530864,4.7171974522293,8.42785445420326,5.5,7.80077369439072,4.98214285714286,6.245,4.69859813084112,5.25982532751092,5.03580562659847,3.90939597315436,5.55191256830601,7.78609625668449,3.10108303249097,7.84752475247525,3.77545691906005,5.17836257309941,3.35365853658537,4.26147704590818,7.66893424036281,9.60742705570292,5.96057347670251,4.42534722222222,5.65426695842451,6.73521126760563,6.08888888888889,3.82374100719424,5.22397476340694,1.53275109170306,2.45939086294416,5.77797833935018,5.31201550387597,5.02164502164502,3.66304347826087,6.43701799485861,8.68627450980392,4.61170212765957,5.4034749034749,4.34455958549223,5.01848049281314,4.95133819951338,6.76038338658147,4.86313465783664,7.02610966057441,4.21898928024502,4.51146384479718,5.55473098330241,5.40629095674967,6.50802139037433,1,7.58595641646489,7.63483146067416,5.2603550295858,5.34375,7.16576086956522,5.27797833935018,4.86029411764706,5.67137809187279,6.23461538461538,5.42284569138277,5.27922077922078,6.61892583120205,4.4468546637744,7.66666666666667,3.50335570469799,9.03468208092486,8.11407766990291,6.00696055684455,6.67048710601719,6.86161879895561,9.06477732793522,6.63582677165354,7.83870967741935,5.00804289544236,6.88421052631579,6.81736281736282,4.74213836477987,5.07466666666667,5.53973509933775,5.28043143297381,3.35913312693498,3.20743034055728,5.81287726358149,4.29411764705882,3.69971264367816,2.10864197530864,6.5872641509434,5.26143790849673,5.49211908931699,6.03059273422562,5.02484472049689,4.286701208981,5.72777777777778,6.52459016393443,4.66059602649007,2.29545454545455,6.26929982046679,2.04128440366972,4.54126213592233,5.64899451553931,6.26600284495021,7.92214111922141,6.91509433962264,7.22736030828516,8.05357142857143,4.39556377079482,4.55701754385965,4.4104347826087,5.49602122015915,6.51932367149758,4.05590062111801,5.97932816537468,5.57352941176471,1,5.12067260138477,5.31781701444623,7.21409214092141,7.25666666666667,1.58249158249158,6.11611374407583,6.06015037593985,5.79088471849866,5.26860841423948,6.77057356608479,7.27258320126783,1,4.45168067226891,6.85614849187935,2.58771929824561,5.64957264957265,3.16236162361624,5.00811688311688,6.84223300970874,6.71047619047619,5.61165048543689,5.81987577639752,9.19603524229075,13.4307304785894,5.44731182795699,6.66576454668471,4.20253164556962,3.36660929432014,2,2.76771653543307,4.45772058823529,6.30989583333333,3.78543307086614,6.38362068965517,6.26327944572748,6.65369649805448,8.91960784313726,4.19438877755511,7.49405772495756,9.59079283887468,6.5607476635514,3.17549668874172,5.2384,4.72072072072072,4.665,5.40865384615385,5,3.44604316546763,5.95299837925446,5.29471544715447,6.37731481481481,9.62439024390244,7.06036745406824,5.33566433566434,5.11278195488722,6.81875,5.46632124352332,3.7811320754717,5.51171171171171,2.23515439429929,7.04716981132075,9.14383561643836,5.902394106814,7.91735537190083,6.06191369606004,7.05731225296443,5.79785604900459,6.59009900990099,8.58842975206612,6.71542553191489,4.38691588785047,7.31944444444444,4.98484848484848,5.55555555555556,6.84698275862069,4.296,3.9358059914408,3.95573440643863,5.52286282306163,4.48251748251748,7.13176470588235,4.99792531120332,4.3168044077135,4.46136865342163,5.59465478841871,6.26239067055394,5.59733333333333,6.87158469945355,6.97791798107256,5.74835886214442,5.36453201970443,5.5275,5.72203389830509,6.57109004739336,5.46210268948655,4.72535211267606,5.847533632287,1,4.85576923076923,6.43065693430657,5.72357723577236,5.84855769230769,5.01627486437613,4.18153846153846,10.5619469026549,2.71341463414634,2.31578947368421,6.15966386554622,1,1,4.78846153846154,5.96782841823056,6.40391459074733,7.91830985915493,6.02608695652174,7.36065573770492,5.57570093457944,6.79316888045541,6.77204301075269,3.53921568627451,5.29861111111111,7.5,7.26016260162602,4.86923076923077,6.46172248803828,6.42965367965368,4.69590643274854,1.15909090909091,2.125,2.35380835380835,1,10.4725848563969,7.37046004842615,5.29899497487437,8.25333333333333,5.00579710144928,6.15159574468085,7.31675874769797,4.46675900277008,7.43956043956044,9.19070904645477,3.5518018018018,7.46546546546547,6.14090019569472,5.08247422680412,7.28285077951002,8.89622641509434,3.625,3.7834008097166,7.23986486486486,7.0734126984127,3.18695652173913,5.07594936708861,8.02191780821918,6.32132424537488,6.60533333333333,6.23076923076923,4.685,7.83221476510067,6.05228758169935,5.07581967213115,7.2314606741573,4.57751937984496,5.91370558375634,6.1996879875195,6.90140845070423,7.1644204851752,5.61975308641975,5.44134078212291,5.85851318944844,6.55769230769231,4.98770491803279,7.83743842364532,6.03370786516854,7.02261306532663,8.55606407322654,6.52448979591837,4.94134078212291,5.86493506493507,5.07269155206287,5.95592286501377,5.65060240963855,6.09273182957393,11.6056910569106,4.54487179487179,6.62437395659432,4.85641891891892,5.38208409506398,5.22434367541766,3.44021739130435,5.52553191489362,6.15926892950392,4.7534983853606,8.080078125,5.97478991596639,4.7747572815534,5.14953271028037,8.49477351916376,5.60215053763441,5.69253731343284,6.71844660194175,7.43526170798898,5.77824267782427,2.35593220338983,3.47741935483871,7.23291139240506,6.22128378378378,5.96173469387755,6.3008356545961,5.74103585657371,7.0906200317965,6.23251748251748,7.08149779735683,6.48854961832061,3.29411764705882,4.80492424242424,6.72021660649819,6.58457711442786,8.5025,7.08771929824561,5.4507299270073,6.93273542600897,5.95141700404858,4.84395604395604,7.76819923371647,5.46351931330472,5.21314387211368,1.9,4.14596273291925,3.8125,7.92757009345794,1.95359628770302,4.03951367781155,5.78981937602627,2.4972191323693,5.64634146341463,5.97900262467192,5.22682926829268,8.07647058823529,1,6,5.81073446327684,4.42884250474383,6.62528735632184,5.94410876132931,6.62738853503185,6.59692898272553,5.87686567164179,5.07777777777778,6.90148698884758,5.37434554973822,6.44657534246575,6.66666666666667,5.60217983651226,8.44705882352941,6.07294117647059,5.67311072056239,3.79611650485437,4.84889643463497,2.07481296758105,1,10.6174698795181,1.5979381443299],"text":["Mean_Morphology_Major_Axis_Length:  62.662900<br />segmentation_metric:   6.658730<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  75.708204<br />segmentation_metric:   9.210826<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  68.875384<br />segmentation_metric:   5.104369<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  76.218173<br />segmentation_metric:   5.042440<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  55.303985<br />segmentation_metric:   4.720859<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.521171<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.244785<br />segmentation_metric:   3.632479<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  80.697714<br />segmentation_metric:   6.005464<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 108.996737<br />segmentation_metric:   5.284232<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 122.700155<br />segmentation_metric:  16.703704<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  23.311408<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  97.364557<br />segmentation_metric:   5.792279<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  57.289245<br />segmentation_metric:   3.046346<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.815541<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  82.859030<br />segmentation_metric:   4.530864<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.988386<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 169.902203<br />segmentation_metric:   6.739130<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  75.173894<br />segmentation_metric:   5.689034<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 119.146322<br />segmentation_metric:  26.240385<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  96.285102<br />segmentation_metric:   5.828697<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  62.360590<br />segmentation_metric:   4.551852<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 169.526705<br />segmentation_metric:  29.102646<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.699408<br />segmentation_metric:   2.074176<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  15.439212<br />segmentation_metric:   3.411765<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  88.432224<br />segmentation_metric:   6.781155<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.636288<br />segmentation_metric:   1.842697<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.275309<br />segmentation_metric:   3.827974<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 107.364133<br />segmentation_metric:  10.606383<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:   2.309401<br />segmentation_metric:   0.300000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.920321<br />segmentation_metric:   5.009980<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.792263<br />segmentation_metric:   5.132616<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  60.990996<br />segmentation_metric:   3.041885<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  97.859763<br />segmentation_metric:   6.195918<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.978183<br />segmentation_metric:   6.100529<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  72.344510<br />segmentation_metric:   4.771375<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.189497<br />segmentation_metric:   6.789474<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.100462<br />segmentation_metric:   5.246704<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  54.255068<br />segmentation_metric:   1.292880<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.844529<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.996468<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  55.292222<br />segmentation_metric:   3.649895<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.702156<br />segmentation_metric:   3.845109<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.444316<br />segmentation_metric:   5.349398<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.057436<br />segmentation_metric:   3.657620<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  34.584032<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.112846<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.876753<br />segmentation_metric:   4.267925<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.111274<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.869197<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.401122<br />segmentation_metric:   5.607639<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.496569<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  23.405226<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  23.314849<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  16.548924<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.878205<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  39.050925<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.426133<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.536145<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.614943<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  70.101859<br />segmentation_metric:   4.724138<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.795863<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.463222<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.072486<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.083788<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.941783<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  72.185281<br />segmentation_metric:   3.363229<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.757023<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.534861<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.587318<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  77.296704<br />segmentation_metric:   4.831858<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  38.092072<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.639857<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  19.729055<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.819889<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  68.780521<br />segmentation_metric:   4.358714<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.462099<br />segmentation_metric:   3.685315<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  56.147388<br />segmentation_metric:   3.872549<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.792018<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  34.255558<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  59.646641<br />segmentation_metric:   5.524096<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  58.130910<br />segmentation_metric:   3.151042<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.344104<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.269451<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.598661<br />segmentation_metric:   5.745050<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.815105<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  57.953561<br />segmentation_metric:   4.583120<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.184247<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.565919<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.153344<br />segmentation_metric:   4.912536<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 101.433352<br />segmentation_metric:   6.965269<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  59.972146<br />segmentation_metric:   5.109551<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.668826<br />segmentation_metric:   8.071053<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 118.652295<br />segmentation_metric:   2.977695<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  86.559026<br />segmentation_metric:   5.224944<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  57.904766<br />segmentation_metric:   1.551859<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 110.841383<br />segmentation_metric:   6.078212<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 125.029688<br />segmentation_metric:   6.594595<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.782398<br />segmentation_metric:   5.978587<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  81.129330<br />segmentation_metric:   4.768212<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.825814<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  70.572984<br />segmentation_metric:   6.981172<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.330603<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.994101<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  54.828491<br />segmentation_metric:   5.441520<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.807629<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  99.526327<br />segmentation_metric:   9.794682<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.283531<br />segmentation_metric:   6.085526<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  63.063841<br />segmentation_metric:   6.282258<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  43.591925<br />segmentation_metric:   4.131148<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  67.369183<br />segmentation_metric:   4.551724<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  50.364284<br />segmentation_metric:   3.784916<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.791849<br />segmentation_metric:   8.046823<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  48.683255<br />segmentation_metric:   1.521605<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.371986<br />segmentation_metric:   5.895062<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  47.260351<br />segmentation_metric:   5.330909<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  98.274166<br />segmentation_metric:  10.048387<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  67.381067<br />segmentation_metric:   6.051205<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.475140<br />segmentation_metric:   6.768362<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  55.505103<br />segmentation_metric:   4.205805<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.652479<br />segmentation_metric:   4.581121<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  63.126535<br />segmentation_metric:   4.542105<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.794133<br />segmentation_metric:   5.965679<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.676140<br />segmentation_metric:   4.120419<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  68.703276<br />segmentation_metric:   4.567119<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.009861<br />segmentation_metric:   4.519920<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 100.574430<br />segmentation_metric:   7.215078<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 102.536661<br />segmentation_metric:   9.164583<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  70.111697<br />segmentation_metric:   4.654639<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 125.537143<br />segmentation_metric:   5.594524<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  90.190308<br />segmentation_metric:   8.550607<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.423872<br />segmentation_metric:   1.002028<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 112.899115<br />segmentation_metric:   6.266562<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.656520<br />segmentation_metric:   5.532710<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.809717<br />segmentation_metric:   4.689888<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  90.423707<br />segmentation_metric:  73.090909<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.957266<br />segmentation_metric:   5.660099<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 104.992938<br />segmentation_metric:   7.189899<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 113.967162<br />segmentation_metric:   6.843072<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.247847<br />segmentation_metric:   5.111340<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  24.406872<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 132.930909<br />segmentation_metric:  15.729107<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.355039<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  24.809146<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  22.088635<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.471173<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  83.076605<br />segmentation_metric:   7.058511<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  84.694085<br />segmentation_metric:   6.637987<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 185.772187<br />segmentation_metric:   9.096284<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.587603<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.271078<br />segmentation_metric:   5.550617<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.596542<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.492971<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.704939<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.813318<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  64.157390<br />segmentation_metric:   5.675879<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.807049<br />segmentation_metric:   5.572082<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.743545<br />segmentation_metric:   7.360791<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 101.981293<br />segmentation_metric:   4.872340<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 117.760508<br />segmentation_metric:   6.792920<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.477221<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  64.319211<br />segmentation_metric:   4.965438<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  64.379865<br />segmentation_metric:   3.667300<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 124.471381<br />segmentation_metric:   3.604530<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.542805<br />segmentation_metric:   6.818605<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  46.966203<br />segmentation_metric:   3.535849<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 100.001213<br />segmentation_metric:   5.463822<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  98.413083<br />segmentation_metric:   4.231738<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  91.039660<br />segmentation_metric:   5.211494<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  36.039327<br />segmentation_metric:   2.176211<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  48.725068<br />segmentation_metric:   3.521519<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  67.929006<br />segmentation_metric:   5.657080<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.071049<br />segmentation_metric:   6.894040<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  62.295392<br />segmentation_metric:   5.560924<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  61.463272<br />segmentation_metric:   3.790076<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  85.457993<br />segmentation_metric:   6.653226<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  86.305413<br />segmentation_metric:   5.528517<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.441017<br />segmentation_metric:   6.422925<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.252491<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  58.721222<br />segmentation_metric:   4.160550<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.684966<br />segmentation_metric:   6.663793<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.746945<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.598835<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  15.754464<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  16.082283<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 104.771192<br />segmentation_metric:   6.085258<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.779325<br />segmentation_metric:   4.373643<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  99.981156<br />segmentation_metric:   7.119355<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.264871<br />segmentation_metric:   6.184810<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  63.836470<br />segmentation_metric:   3.982379<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.741252<br />segmentation_metric:   7.217582<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  95.698352<br />segmentation_metric:   9.601980<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.173415<br />segmentation_metric:   6.767123<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.436239<br />segmentation_metric:   6.507143<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  65.413969<br />segmentation_metric:   6.191489<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.443521<br />segmentation_metric:   5.491935<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.401308<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.889954<br />segmentation_metric:   5.764045<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  75.684390<br />segmentation_metric:   7.975207<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 102.500444<br />segmentation_metric:   5.074380<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 107.208440<br />segmentation_metric:   9.053908<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.555939<br />segmentation_metric:   5.516484<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.718901<br />segmentation_metric:   7.097436<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  60.778182<br />segmentation_metric:   6.648501<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  67.970973<br />segmentation_metric:   5.405479<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  16.732423<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.393635<br />segmentation_metric:   7.907090<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  95.068914<br />segmentation_metric:   7.137850<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 108.910051<br />segmentation_metric:   6.204604<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.163384<br />segmentation_metric:   5.310219<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.623086<br />segmentation_metric:   5.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.022072<br />segmentation_metric:   5.419525<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 122.448530<br />segmentation_metric:   3.336705<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.646089<br />segmentation_metric:   5.196203<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  78.050279<br />segmentation_metric:   6.703704<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  80.716372<br />segmentation_metric:   5.888235<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 106.129786<br />segmentation_metric:   5.787743<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  85.445496<br />segmentation_metric:   4.115079<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 101.427002<br />segmentation_metric:   7.696970<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  86.796031<br />segmentation_metric:   5.392092<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.914843<br />segmentation_metric:   4.699454<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 103.265644<br />segmentation_metric:   4.569579<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.615859<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.467223<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.611775<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.145254<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.363024<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.042084<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.105512<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.184467<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.786425<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  16.522347<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   8.958557<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.049968<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.746552<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.637242<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.673398<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.675591<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  32.373783<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.900327<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.314837<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.339307<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.320031<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.126821<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.133991<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  49.171187<br />segmentation_metric:   3.982301<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.431641<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.274911<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.265500<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.889665<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  22.667679<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.954662<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.652120<br />segmentation_metric:  14.154506<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.343989<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.829067<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 106.868485<br />segmentation_metric:   9.614796<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.372424<br />segmentation_metric:   4.981395<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  62.931845<br />segmentation_metric:   3.747748<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 130.846979<br />segmentation_metric:   4.599444<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  53.604163<br />segmentation_metric:   4.921260<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.554512<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  67.191031<br />segmentation_metric:   7.097179<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  63.673007<br />segmentation_metric:   4.178649<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.025839<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.917559<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.338737<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  20.163759<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.777969<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.513532<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.824011<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  38.475169<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.524665<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.542113<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.788590<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  65.439738<br />segmentation_metric:   5.064897<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.596440<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  32.715587<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  32.402120<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.872537<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 202.792183<br />segmentation_metric:   5.629747<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.450647<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 146.902952<br />segmentation_metric:   8.714286<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  48.621907<br />segmentation_metric:   1.427152<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.604807<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  83.061804<br />segmentation_metric:   7.590588<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  22.845915<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.766311<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.245104<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.250854<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  68.883426<br />segmentation_metric:   7.496333<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.376277<br />segmentation_metric:   6.868293<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.729367<br />segmentation_metric:   4.616981<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.691414<br />segmentation_metric:   6.614035<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  99.369656<br />segmentation_metric:   8.466667<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.415037<br />segmentation_metric:  10.404348<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.305923<br />segmentation_metric:   6.015000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 119.322520<br />segmentation_metric:   4.811966<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.691152<br />segmentation_metric:   7.249501<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.267647<br />segmentation_metric:   7.069264<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.022833<br />segmentation_metric:   5.436573<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.193293<br />segmentation_metric:   7.223958<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.665574<br />segmentation_metric:   6.859122<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  36.800288<br />segmentation_metric:   2.127371<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.688059<br />segmentation_metric:   7.060855<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 109.672053<br />segmentation_metric:   7.105398<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.382844<br />segmentation_metric:   8.322515<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.351944<br />segmentation_metric:   3.330724<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.567432<br />segmentation_metric:   5.303665<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.543573<br />segmentation_metric:   3.578378<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  56.951448<br />segmentation_metric:   4.258383<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  32.256891<br />segmentation_metric:   7.437500<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.600720<br />segmentation_metric:   6.596859<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 141.778250<br />segmentation_metric:   8.107143<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 117.442554<br />segmentation_metric:   4.099200<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.482375<br />segmentation_metric:   5.330211<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.580865<br />segmentation_metric:   2.942857<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  52.746119<br />segmentation_metric:   3.616592<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.023475<br />segmentation_metric:   4.269321<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.819514<br />segmentation_metric:   4.901193<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  30.804028<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.979252<br />segmentation_metric:   5.728102<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 163.086782<br />segmentation_metric:   7.274956<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.932901<br />segmentation_metric:   4.249554<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 135.221914<br />segmentation_metric:   6.434057<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.333221<br />segmentation_metric:   6.643059<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  99.648037<br />segmentation_metric:   6.679646<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 110.526700<br />segmentation_metric:   7.797721<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 108.036401<br />segmentation_metric:   7.123684<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  31.485500<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.373892<br />segmentation_metric:   6.546099<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.290501<br />segmentation_metric:   7.384211<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.586002<br />segmentation_metric:   4.287561<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 128.551294<br />segmentation_metric:   5.803625<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.039554<br />segmentation_metric:   8.205279<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.365175<br />segmentation_metric:   6.113953<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.974109<br />segmentation_metric:   5.706849<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.277234<br />segmentation_metric:   4.796992<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.823339<br />segmentation_metric:   5.632231<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.314593<br />segmentation_metric:   4.972581<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.847611<br />segmentation_metric:   5.919588<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  56.653155<br />segmentation_metric:   4.143590<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  52.362605<br />segmentation_metric:   7.185455<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.032904<br />segmentation_metric:   4.703526<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.511416<br />segmentation_metric:   6.624294<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  73.796934<br />segmentation_metric:   5.308511<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  48.995622<br />segmentation_metric:   3.858934<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.195192<br />segmentation_metric:   4.713147<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 149.096372<br />segmentation_metric:   8.329966<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 113.113653<br />segmentation_metric:   6.206235<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.919680<br />segmentation_metric:   6.652778<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.297555<br />segmentation_metric:   3.600000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.143612<br />segmentation_metric:   5.092564<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.277815<br />segmentation_metric:   5.626984<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  24.256879<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  31.626449<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.511940<br />segmentation_metric:   5.800830<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  26.477244<br />segmentation_metric:   1.194805<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.790856<br />segmentation_metric:   5.026408<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 109.774822<br />segmentation_metric:   4.520067<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 114.314109<br />segmentation_metric:   6.260674<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.049329<br />segmentation_metric:   3.413655<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  26.677659<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  76.845219<br />segmentation_metric:   6.284065<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 101.063865<br />segmentation_metric:   6.017204<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  71.763786<br />segmentation_metric:   5.978667<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  73.838978<br />segmentation_metric:   5.185596<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.390842<br />segmentation_metric:   5.626556<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.045591<br />segmentation_metric:   3.935583<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.040785<br />segmentation_metric:   5.438642<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 134.392896<br />segmentation_metric:   6.201014<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.440829<br />segmentation_metric:   3.297753<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.731134<br />segmentation_metric:   3.195266<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.260593<br />segmentation_metric:   5.434896<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.686700<br />segmentation_metric:   2.442886<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 141.220193<br />segmentation_metric:   8.027823<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 161.484434<br />segmentation_metric:   4.727932<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 107.264790<br />segmentation_metric:   6.900609<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  50.408217<br />segmentation_metric:   3.251969<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  90.743506<br />segmentation_metric:   6.759554<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.342420<br />segmentation_metric:   3.916877<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.009027<br />segmentation_metric:   5.467066<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  46.292882<br />segmentation_metric:   2.382090<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.433784<br />segmentation_metric:   4.420345<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  95.309021<br />segmentation_metric:   6.335060<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 115.368734<br />segmentation_metric:   5.733102<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  55.610943<br />segmentation_metric:   4.089202<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  36.726090<br />segmentation_metric:   2.324713<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.687921<br />segmentation_metric:   4.837370<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.927741<br />segmentation_metric:   5.115324<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 109.824791<br />segmentation_metric:   5.939866<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  53.050581<br />segmentation_metric:   5.079545<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  24.734042<br />segmentation_metric:   3.151079<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.064560<br />segmentation_metric:   6.572890<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.386823<br />segmentation_metric:   5.075431<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 117.033714<br />segmentation_metric:  10.914336<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.505781<br />segmentation_metric:   5.407609<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.840897<br />segmentation_metric:   6.069652<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.323053<br />segmentation_metric:   9.217241<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  43.889012<br />segmentation_metric:   3.912773<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.245600<br />segmentation_metric:   4.688406<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  66.805929<br />segmentation_metric:   5.917112<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  41.712536<br />segmentation_metric:   2.800000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.104795<br />segmentation_metric:   5.487923<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.657414<br />segmentation_metric:   5.314121<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 115.522753<br />segmentation_metric:   5.614964<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.345724<br />segmentation_metric:   3.934959<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.207632<br />segmentation_metric:   6.090909<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  66.875321<br />segmentation_metric:   5.576754<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 108.358606<br />segmentation_metric:   6.520576<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 125.818564<br />segmentation_metric:   8.562863<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 119.697280<br />segmentation_metric:   5.593137<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  81.744232<br />segmentation_metric:   6.237537<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 298.352120<br />segmentation_metric:   7.024850<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.445285<br />segmentation_metric:   8.561404<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 103.336784<br />segmentation_metric:   6.923937<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  60.433294<br />segmentation_metric:   6.444444<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 119.297230<br />segmentation_metric:   6.929619<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.517890<br />segmentation_metric:   5.504184<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 104.301922<br />segmentation_metric:   5.039539<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.456525<br />segmentation_metric:   6.591703<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.899916<br />segmentation_metric:   4.937984<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.757056<br />segmentation_metric:   6.759375<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.671905<br />segmentation_metric:   4.934066<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  31.490607<br />segmentation_metric:   1.888000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 108.733937<br />segmentation_metric:   4.604167<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.701636<br />segmentation_metric:   6.153680<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.938040<br />segmentation_metric:   5.927875<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.088392<br />segmentation_metric:   4.449749<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.704410<br />segmentation_metric:   5.248731<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.299041<br />segmentation_metric:   7.398524<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.537207<br />segmentation_metric:   6.050360<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.028359<br />segmentation_metric:   6.284058<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  29.760363<br />segmentation_metric:  10.894737<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  33.971820<br />segmentation_metric:   1.533333<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.991565<br />segmentation_metric:   8.031325<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 110.626617<br />segmentation_metric:   7.850575<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.894001<br />segmentation_metric:   5.318059<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  22.198054<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  76.979478<br />segmentation_metric:   6.262857<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  59.949698<br />segmentation_metric:   5.685801<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.577992<br />segmentation_metric:   4.734463<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 135.654564<br />segmentation_metric:   4.804659<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.432704<br />segmentation_metric:   4.691466<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.882599<br />segmentation_metric:   4.974468<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  71.043184<br />segmentation_metric:   7.311927<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.911098<br />segmentation_metric:   5.495913<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  54.096871<br />segmentation_metric:   5.711765<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.043938<br />segmentation_metric:   4.903930<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.571940<br />segmentation_metric:   6.392647<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  21.211924<br />segmentation_metric:   1.986111<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:   4.381780<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  33.749418<br />segmentation_metric:   4.868421<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 101.129534<br />segmentation_metric:   5.854902<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.059557<br />segmentation_metric:   6.361765<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.088173<br />segmentation_metric:   5.982609<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 104.814363<br />segmentation_metric:   6.643275<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.949948<br />segmentation_metric:   8.419558<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  95.329411<br />segmentation_metric:   6.147445<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.547077<br />segmentation_metric:   5.318087<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 103.788165<br />segmentation_metric:   9.289474<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.932107<br />segmentation_metric:   7.507853<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  60.103907<br />segmentation_metric:   7.423488<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 134.783439<br />segmentation_metric:   6.395397<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 110.838568<br />segmentation_metric:   6.553333<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 202.121642<br />segmentation_metric:  11.921105<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  73.240070<br />segmentation_metric:   6.147668<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.368943<br />segmentation_metric:   8.177945<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  61.541264<br />segmentation_metric:   4.619444<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.871332<br />segmentation_metric:   6.234266<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  99.845595<br />segmentation_metric:   6.328125<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  53.294112<br />segmentation_metric:   5.560656<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.485498<br />segmentation_metric:   4.420131<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  75.673303<br />segmentation_metric:   5.783562<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.409987<br />segmentation_metric:   5.937063<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  60.455513<br />segmentation_metric:   6.864307<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.196560<br />segmentation_metric:   4.943633<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  65.628944<br />segmentation_metric:   7.895161<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.548138<br />segmentation_metric:   3.893112<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 112.458991<br />segmentation_metric:   7.292484<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  56.357687<br />segmentation_metric:   5.177156<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  25.857714<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.863873<br />segmentation_metric:   9.469101<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  60.472351<br />segmentation_metric:   7.557047<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  54.178893<br />segmentation_metric:   5.112329<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  28.301326<br />segmentation_metric:   2.173611<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 120.182126<br />segmentation_metric:   7.873368<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.847405<br />segmentation_metric:   5.515924<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 105.451402<br />segmentation_metric:   3.545833<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 122.503659<br />segmentation_metric:   4.981037<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 124.929742<br />segmentation_metric:   6.786765<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  59.091274<br />segmentation_metric:   2.291016<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  49.972489<br />segmentation_metric:   5.157459<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.190181<br />segmentation_metric:   4.769126<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  50.240308<br />segmentation_metric:  32.964286<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 104.147337<br />segmentation_metric:   6.793296<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 105.471486<br />segmentation_metric:   6.408257<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.566481<br />segmentation_metric:   5.349920<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.222662<br />segmentation_metric:   5.149057<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.842561<br />segmentation_metric:   4.588323<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.364664<br />segmentation_metric:   4.317195<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  70.154092<br />segmentation_metric:   6.577670<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.096874<br />segmentation_metric:   4.299233<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.484944<br />segmentation_metric:   3.836667<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  32.379754<br />segmentation_metric:   2.462745<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.311531<br />segmentation_metric:   5.463840<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  45.855865<br />segmentation_metric:   3.192090<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 186.090782<br />segmentation_metric:   7.755167<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.562386<br />segmentation_metric:   5.788599<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.590110<br />segmentation_metric:   5.842222<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  56.998027<br />segmentation_metric:   3.841649<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 133.566216<br />segmentation_metric:   4.856838<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.295719<br />segmentation_metric:  11.426230<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.935794<br />segmentation_metric:   7.716814<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.999509<br />segmentation_metric:   6.885027<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.570810<br />segmentation_metric:   5.943320<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  58.641476<br />segmentation_metric:   5.201087<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.398933<br />segmentation_metric:   6.099057<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 116.552201<br />segmentation_metric:   6.021277<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  57.212007<br />segmentation_metric:   4.904393<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 164.659819<br />segmentation_metric:   5.092593<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.769166<br />segmentation_metric:   8.408898<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  85.832027<br />segmentation_metric:   7.320652<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.592093<br />segmentation_metric:   7.460560<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.119928<br />segmentation_metric:   4.867850<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.173096<br />segmentation_metric:   6.511013<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.339006<br />segmentation_metric:   3.931159<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  22.728097<br />segmentation_metric:   2.681416<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  58.109120<br />segmentation_metric:   5.701818<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.874502<br />segmentation_metric:   5.322335<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.089949<br />segmentation_metric:   7.105042<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.483310<br />segmentation_metric:   6.983607<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.595526<br />segmentation_metric:   9.420455<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.931624<br />segmentation_metric:  21.411111<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   8.775421<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 109.516451<br />segmentation_metric:   8.259101<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.921606<br />segmentation_metric:   4.353383<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.900621<br />segmentation_metric:   9.082789<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.298325<br />segmentation_metric:   6.044094<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  69.337619<br />segmentation_metric:   5.745182<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  49.764184<br />segmentation_metric:   4.411765<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.585696<br />segmentation_metric:   4.888636<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.131861<br />segmentation_metric:   5.691729<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   9.459173<br />segmentation_metric:   2.300000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  51.965006<br />segmentation_metric:   4.908497<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.297854<br />segmentation_metric:   6.954887<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.383975<br />segmentation_metric:   6.443082<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.366514<br />segmentation_metric:   9.147679<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 108.048627<br />segmentation_metric:   6.589928<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 106.077847<br />segmentation_metric:   6.873504<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.361715<br />segmentation_metric:   7.136095<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.275331<br />segmentation_metric:   7.327456<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.226853<br />segmentation_metric:   7.655827<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.525030<br />segmentation_metric:   7.675847<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.443610<br />segmentation_metric:   7.649798<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  56.261012<br />segmentation_metric:   5.696429<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  61.735220<br />segmentation_metric:   6.261398<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.491707<br />segmentation_metric:   6.141333<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.006661<br />segmentation_metric:   5.211735<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  84.669630<br />segmentation_metric:   6.563715<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.115322<br />segmentation_metric:   7.422274<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.727569<br />segmentation_metric:   4.632850<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  36.089035<br />segmentation_metric:   1.109005<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 138.043802<br />segmentation_metric:   6.130612<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.221204<br />segmentation_metric:   5.975655<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.845516<br />segmentation_metric:   8.045802<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.460621<br />segmentation_metric:   5.268595<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  93.024347<br />segmentation_metric:   4.480609<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  65.345246<br />segmentation_metric:   5.425287<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.364289<br />segmentation_metric:   7.398210<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.369917<br />segmentation_metric:   5.736501<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  75.630630<br />segmentation_metric:   6.798627<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.180836<br />segmentation_metric:   4.667622<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.259084<br />segmentation_metric:   7.069869<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  38.152751<br />segmentation_metric:   5.238462<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 120.748549<br />segmentation_metric:   6.056911<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 107.781535<br />segmentation_metric:   5.462745<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.522868<br />segmentation_metric:   4.392857<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 119.839093<br />segmentation_metric:   4.395939<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 110.380646<br />segmentation_metric:   6.423445<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 118.407122<br />segmentation_metric:   3.557530<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 115.224589<br />segmentation_metric:   4.425197<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 103.072968<br />segmentation_metric:   5.533608<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  62.601796<br />segmentation_metric:   5.604712<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.509139<br />segmentation_metric:   9.657277<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.813583<br />segmentation_metric:   6.369330<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.595615<br />segmentation_metric:   6.661058<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  60.454356<br />segmentation_metric:   6.141538<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.454302<br />segmentation_metric:   5.795276<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  72.665761<br />segmentation_metric:   5.345382<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.601920<br />segmentation_metric:   5.440181<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.788090<br />segmentation_metric:   5.109865<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.868246<br />segmentation_metric:   7.820513<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.373413<br />segmentation_metric:   5.863388<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 101.711733<br />segmentation_metric:   5.559509<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.302180<br />segmentation_metric:   6.134490<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 121.475917<br />segmentation_metric:   5.292453<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.773674<br />segmentation_metric:   6.945619<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.881217<br />segmentation_metric:   4.073756<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  98.870728<br />segmentation_metric:   6.584746<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.591027<br />segmentation_metric:   6.316327<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.452034<br />segmentation_metric:   5.105477<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  62.018884<br />segmentation_metric:   5.054264<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  15.704789<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 231.725464<br />segmentation_metric: 186.758065<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  85.956175<br />segmentation_metric:   7.175879<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  60.347201<br />segmentation_metric:   3.063011<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 142.950117<br />segmentation_metric:   4.969088<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.932114<br />segmentation_metric:   7.232346<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  40.777838<br />segmentation_metric:   4.214876<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  17.537149<br />segmentation_metric:   4.243243<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 120.959258<br />segmentation_metric:   5.514469<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.393618<br />segmentation_metric:   5.779956<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.201190<br />segmentation_metric:   7.876289<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  13.443028<br />segmentation_metric:   1.136364<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.299962<br />segmentation_metric:   5.712500<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.520765<br />segmentation_metric:   7.986877<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  27.447777<br />segmentation_metric:   1.587963<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  23.935480<br />segmentation_metric:   1.790244<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  85.724215<br />segmentation_metric:   4.695210<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.264064<br />segmentation_metric:   6.231579<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.554523<br />segmentation_metric:   3.846682<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  82.727228<br />segmentation_metric:   8.131519<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.312804<br />segmentation_metric:   5.208617<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  97.277040<br />segmentation_metric:   5.184322<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 106.750523<br />segmentation_metric:   5.497487<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.185573<br />segmentation_metric:   5.829684<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  34.524300<br />segmentation_metric:   2.260714<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  71.275047<br />segmentation_metric:   5.606719<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.010372<br />segmentation_metric:   4.665192<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.499145<br />segmentation_metric:   7.231672<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.886332<br />segmentation_metric:   7.417553<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.684707<br />segmentation_metric:   5.502128<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 119.965626<br />segmentation_metric:   6.638083<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 155.689920<br />segmentation_metric:   6.957746<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  52.660252<br />segmentation_metric:   3.186944<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  93.649229<br />segmentation_metric:   4.942053<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.682817<br />segmentation_metric:   4.486811<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  55.832597<br />segmentation_metric:   5.662338<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 102.007689<br />segmentation_metric:  10.322485<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.114027<br />segmentation_metric:   4.358757<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.980905<br />segmentation_metric:   4.727273<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  51.358027<br />segmentation_metric:   1.889855<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  46.089095<br />segmentation_metric:   1.908642<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.575365<br />segmentation_metric:   4.717197<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 144.143772<br />segmentation_metric:   8.427854<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  13.929185<br />segmentation_metric:   5.500000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 106.376194<br />segmentation_metric:   7.800774<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  73.540315<br />segmentation_metric:   4.982143<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.879657<br />segmentation_metric:   6.245000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  62.920053<br />segmentation_metric:   4.698598<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.459707<br />segmentation_metric:   5.259825<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.187486<br />segmentation_metric:   5.035806<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  58.096059<br />segmentation_metric:   3.909396<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.574019<br />segmentation_metric:   5.551913<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  85.422710<br />segmentation_metric:   7.786096<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  46.115741<br />segmentation_metric:   3.101083<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.752227<br />segmentation_metric:   7.847525<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  51.828123<br />segmentation_metric:   3.775457<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.404029<br />segmentation_metric:   5.178363<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  47.771933<br />segmentation_metric:   3.353659<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.043044<br />segmentation_metric:   4.261477<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  87.299344<br />segmentation_metric:   7.668934<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 115.747868<br />segmentation_metric:   9.607427<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  95.955867<br />segmentation_metric:   5.960573<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.511760<br />segmentation_metric:   4.425347<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 112.677759<br />segmentation_metric:   5.654267<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 113.645138<br />segmentation_metric:   6.735211<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.243699<br />segmentation_metric:   6.088889<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  39.035929<br />segmentation_metric:   3.823741<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.348940<br />segmentation_metric:   5.223975<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  28.653026<br />segmentation_metric:   1.532751<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  44.966892<br />segmentation_metric:   2.459391<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.449491<br />segmentation_metric:   5.777978<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 120.510941<br />segmentation_metric:   5.312016<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  63.234133<br />segmentation_metric:   5.021645<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  32.823624<br />segmentation_metric:   3.663043<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.502924<br />segmentation_metric:   6.437018<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.202042<br />segmentation_metric:   8.686275<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  45.540550<br />segmentation_metric:   4.611702<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 121.582442<br />segmentation_metric:   5.403475<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.686746<br />segmentation_metric:   4.344560<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.262697<br />segmentation_metric:   5.018480<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.683526<br />segmentation_metric:   4.951338<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  62.205325<br />segmentation_metric:   6.760383<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  62.700125<br />segmentation_metric:   4.863135<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.061094<br />segmentation_metric:   7.026110<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  95.836039<br />segmentation_metric:   4.218989<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  74.111092<br />segmentation_metric:   4.511464<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.338506<br />segmentation_metric:   5.554731<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 188.034576<br />segmentation_metric:   5.406291<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  79.612456<br />segmentation_metric:   6.508021<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  15.945265<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.379078<br />segmentation_metric:   7.585956<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  86.333623<br />segmentation_metric:   7.634831<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.824524<br />segmentation_metric:   5.260355<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  58.060036<br />segmentation_metric:   5.343750<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.107794<br />segmentation_metric:   7.165761<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 110.331552<br />segmentation_metric:   5.277978<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.787806<br />segmentation_metric:   4.860294<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.024583<br />segmentation_metric:   5.671378<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  79.056164<br />segmentation_metric:   6.234615<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  96.503545<br />segmentation_metric:   5.422846<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  62.258274<br />segmentation_metric:   5.279221<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.793116<br />segmentation_metric:   6.618926<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.498695<br />segmentation_metric:   4.446855<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.245919<br />segmentation_metric:   7.666667<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  35.442246<br />segmentation_metric:   3.503356<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  68.981981<br />segmentation_metric:   9.034682<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  84.942279<br />segmentation_metric:   8.114078<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.282884<br />segmentation_metric:   6.006961<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.002919<br />segmentation_metric:   6.670487<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 118.656636<br />segmentation_metric:   6.861619<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 127.843886<br />segmentation_metric:   9.064777<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.781480<br />segmentation_metric:   6.635827<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.028668<br />segmentation_metric:   7.838710<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.036073<br />segmentation_metric:   5.008043<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  79.163562<br />segmentation_metric:   6.884211<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 179.403370<br />segmentation_metric:   6.817363<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.450310<br />segmentation_metric:   4.742138<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.594192<br />segmentation_metric:   5.074667<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.064126<br />segmentation_metric:   5.539735<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 104.644991<br />segmentation_metric:   5.280431<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.830975<br />segmentation_metric:   3.359133<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  52.152452<br />segmentation_metric:   3.207430<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  74.187840<br />segmentation_metric:   5.812877<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.628897<br />segmentation_metric:   4.294118<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.575426<br />segmentation_metric:   3.699713<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.839672<br />segmentation_metric:   2.108642<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.270128<br />segmentation_metric:   6.587264<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 120.756487<br />segmentation_metric:   5.261438<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  96.520025<br />segmentation_metric:   5.492119<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  97.375860<br />segmentation_metric:   6.030593<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.818911<br />segmentation_metric:   5.024845<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.165726<br />segmentation_metric:   4.286701<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 106.116549<br />segmentation_metric:   5.727778<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.970682<br />segmentation_metric:   6.524590<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.556386<br />segmentation_metric:   4.660596<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  44.579475<br />segmentation_metric:   2.295455<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  84.519468<br />segmentation_metric:   6.269300<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  25.760280<br />segmentation_metric:   2.041284<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.357637<br />segmentation_metric:   4.541262<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.344001<br />segmentation_metric:   5.648995<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.589493<br />segmentation_metric:   6.266003<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  96.546891<br />segmentation_metric:   7.922141<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.496143<br />segmentation_metric:   6.915094<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  84.137120<br />segmentation_metric:   7.227360<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 136.498913<br />segmentation_metric:   8.053571<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.843063<br />segmentation_metric:   4.395564<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  78.861324<br />segmentation_metric:   4.557018<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.920650<br />segmentation_metric:   4.410435<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  64.135120<br />segmentation_metric:   5.496021<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  98.713106<br />segmentation_metric:   6.519324<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.893713<br />segmentation_metric:   4.055901<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.844624<br />segmentation_metric:   5.979328<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.628182<br />segmentation_metric:   5.573529<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  25.119602<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  98.810441<br />segmentation_metric:   5.120673<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.704258<br />segmentation_metric:   5.317817<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 141.884429<br />segmentation_metric:   7.214092<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 154.390886<br />segmentation_metric:   7.256667<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  32.736416<br />segmentation_metric:   1.582492<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.153601<br />segmentation_metric:   6.116114<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.766245<br />segmentation_metric:   6.060150<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 110.579095<br />segmentation_metric:   5.790885<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  65.363659<br />segmentation_metric:   5.268608<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.500980<br />segmentation_metric:   6.770574<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 127.679877<br />segmentation_metric:   7.272583<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  28.293437<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.709673<br />segmentation_metric:   4.451681<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.005574<br />segmentation_metric:   6.856148<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  29.435155<br />segmentation_metric:   2.587719<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.108042<br />segmentation_metric:   5.649573<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  36.899184<br />segmentation_metric:   3.162362<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.776876<br />segmentation_metric:   5.008117<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.661747<br />segmentation_metric:   6.842233<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.421574<br />segmentation_metric:   6.710476<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.342994<br />segmentation_metric:   5.611650<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  75.119151<br />segmentation_metric:   5.819876<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 106.553581<br />segmentation_metric:   9.196035<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.368807<br />segmentation_metric:  13.430730<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 114.947865<br />segmentation_metric:   5.447312<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.605361<br />segmentation_metric:   6.665765<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.181129<br />segmentation_metric:   4.202532<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.097665<br />segmentation_metric:   3.366609<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  30.876857<br />segmentation_metric:   2.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  32.809796<br />segmentation_metric:   2.767717<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.650574<br />segmentation_metric:   4.457721<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.812871<br />segmentation_metric:   6.309896<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  79.672991<br />segmentation_metric:   3.785433<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.009090<br />segmentation_metric:   6.383621<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  88.244353<br />segmentation_metric:   6.263279<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 155.883144<br />segmentation_metric:   6.653696<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  99.986826<br />segmentation_metric:   8.919608<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 113.451957<br />segmentation_metric:   4.194389<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  89.955324<br />segmentation_metric:   7.494058<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.474474<br />segmentation_metric:   9.590793<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  79.368974<br />segmentation_metric:   6.560748<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 111.893934<br />segmentation_metric:   3.175497<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 180.548804<br />segmentation_metric:   5.238400<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.235213<br />segmentation_metric:   4.720721<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.511055<br />segmentation_metric:   4.665000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 115.993889<br />segmentation_metric:   5.408654<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  63.984682<br />segmentation_metric:   5.000000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  41.764933<br />segmentation_metric:   3.446043<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  88.615386<br />segmentation_metric:   5.952998<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.228300<br />segmentation_metric:   5.294715<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  71.673582<br />segmentation_metric:   6.377315<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 120.211368<br />segmentation_metric:   9.624390<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.311629<br />segmentation_metric:   7.060367<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 130.158557<br />segmentation_metric:   5.335664<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.306605<br />segmentation_metric:   5.112782<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.364847<br />segmentation_metric:   6.818750<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.129230<br />segmentation_metric:   5.466321<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  42.800388<br />segmentation_metric:   3.781132<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  93.332775<br />segmentation_metric:   5.511712<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  41.983345<br />segmentation_metric:   2.235154<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  88.907005<br />segmentation_metric:   7.047170<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.092734<br />segmentation_metric:   9.143836<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  75.400076<br />segmentation_metric:   5.902394<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.952621<br />segmentation_metric:   7.917355<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.547710<br />segmentation_metric:   6.061914<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  90.568403<br />segmentation_metric:   7.057312<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.540759<br />segmentation_metric:   5.797856<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.375966<br />segmentation_metric:   6.590099<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 133.238551<br />segmentation_metric:   8.588430<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.009451<br />segmentation_metric:   6.715426<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 126.222889<br />segmentation_metric:   4.386916<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  65.439493<br />segmentation_metric:   7.319444<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.070311<br />segmentation_metric:   4.984848<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  47.242249<br />segmentation_metric:   5.555556<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.818613<br />segmentation_metric:   6.846983<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.754018<br />segmentation_metric:   4.296000<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.777873<br />segmentation_metric:   3.935806<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 121.075135<br />segmentation_metric:   3.955734<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.437506<br />segmentation_metric:   5.522863<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  44.433152<br />segmentation_metric:   4.482517<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  90.025421<br />segmentation_metric:   7.131765<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.250099<br />segmentation_metric:   4.997925<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  72.666941<br />segmentation_metric:   4.316804<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  61.844945<br />segmentation_metric:   4.461369<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.905949<br />segmentation_metric:   5.594655<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 108.627725<br />segmentation_metric:   6.262391<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  64.202053<br />segmentation_metric:   5.597333<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.025557<br />segmentation_metric:   6.871585<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  64.314308<br />segmentation_metric:   6.977918<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.988349<br />segmentation_metric:   5.748359<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.702948<br />segmentation_metric:   5.364532<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 106.901530<br />segmentation_metric:   5.527500<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  61.229165<br />segmentation_metric:   5.722034<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  73.864427<br />segmentation_metric:   6.571090<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.098712<br />segmentation_metric:   5.462103<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  69.132641<br />segmentation_metric:   4.725352<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  70.242930<br />segmentation_metric:   5.847534<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  15.672786<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.947767<br />segmentation_metric:   4.855769<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  56.524067<br />segmentation_metric:   6.430657<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.195652<br />segmentation_metric:   5.723577<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  70.630223<br />segmentation_metric:   5.848558<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.250758<br />segmentation_metric:   5.016275<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  48.933520<br />segmentation_metric:   4.181538<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 203.809233<br />segmentation_metric:  10.561947<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  38.317551<br />segmentation_metric:   2.713415<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  39.599707<br />segmentation_metric:   2.315789<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.140172<br />segmentation_metric:   6.159664<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  21.861176<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  29.092412<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.024218<br />segmentation_metric:   4.788462<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.880173<br />segmentation_metric:   5.967828<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 140.350598<br />segmentation_metric:   6.403915<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 106.498528<br />segmentation_metric:   7.918310<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  66.882626<br />segmentation_metric:   6.026087<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.527659<br />segmentation_metric:   7.360656<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 120.544718<br />segmentation_metric:   5.575701<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.047159<br />segmentation_metric:   6.793169<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 121.704756<br />segmentation_metric:   6.772043<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.785455<br />segmentation_metric:   3.539216<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  99.617623<br />segmentation_metric:   5.298611<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  97.208979<br />segmentation_metric:   7.500000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.908359<br />segmentation_metric:   7.260163<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  54.121313<br />segmentation_metric:   4.869231<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.826487<br />segmentation_metric:   6.461722<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 107.763659<br />segmentation_metric:   6.429654<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 117.240967<br />segmentation_metric:   4.695906<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:   9.266493<br />segmentation_metric:   1.159091<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:   8.039188<br />segmentation_metric:   2.125000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  57.755163<br />segmentation_metric:   2.353808<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  30.890308<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.526127<br />segmentation_metric:  10.472585<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.311917<br />segmentation_metric:   7.370460<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.783616<br />segmentation_metric:   5.298995<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  92.134579<br />segmentation_metric:   8.253333<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  52.334004<br />segmentation_metric:   5.005797<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.448320<br />segmentation_metric:   6.151596<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.078621<br />segmentation_metric:   7.316759<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  67.257035<br />segmentation_metric:   4.466759<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.076684<br />segmentation_metric:   7.439560<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 112.721355<br />segmentation_metric:   9.190709<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  79.141522<br />segmentation_metric:   3.551802<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.645478<br />segmentation_metric:   7.465465<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  90.535540<br />segmentation_metric:   6.140900<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.423011<br />segmentation_metric:   5.082474<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.808223<br />segmentation_metric:   7.282851<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 101.010005<br />segmentation_metric:   8.896226<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  10.233944<br />segmentation_metric:   3.625000<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  98.926989<br />segmentation_metric:   3.783401<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  61.569824<br />segmentation_metric:   7.239865<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.299853<br />segmentation_metric:   7.073413<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.191485<br />segmentation_metric:   3.186957<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  66.881884<br />segmentation_metric:   5.075949<br />well_name: E09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.109406<br />segmentation_metric:   8.021918<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 277.628708<br />segmentation_metric:   6.321324<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.865374<br />segmentation_metric:   6.605333<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 101.846700<br />segmentation_metric:   6.230769<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.774785<br />segmentation_metric:   4.685000<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.626906<br />segmentation_metric:   7.832215<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 110.431614<br />segmentation_metric:   6.052288<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.772171<br />segmentation_metric:   5.075820<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.393324<br />segmentation_metric:   7.231461<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.111714<br />segmentation_metric:   4.577519<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 113.604464<br />segmentation_metric:   5.913706<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 145.153827<br />segmentation_metric:   6.199688<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.132126<br />segmentation_metric:   6.901408<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  79.209580<br />segmentation_metric:   7.164420<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.860358<br />segmentation_metric:   5.619753<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 111.038349<br />segmentation_metric:   5.441341<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.169625<br />segmentation_metric:   5.858513<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.926821<br />segmentation_metric:   6.557692<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.650853<br />segmentation_metric:   4.987705<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.228736<br />segmentation_metric:   7.837438<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.624090<br />segmentation_metric:   6.033708<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.326109<br />segmentation_metric:   7.022613<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.111053<br />segmentation_metric:   8.556064<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.200313<br />segmentation_metric:   6.524490<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.903885<br />segmentation_metric:   4.941341<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.265303<br />segmentation_metric:   5.864935<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.812948<br />segmentation_metric:   5.072692<br />well_name: E09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.153905<br />segmentation_metric:   5.955923<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  53.853100<br />segmentation_metric:   5.650602<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.270704<br />segmentation_metric:   6.092732<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 190.630321<br />segmentation_metric:  11.605691<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  49.142209<br />segmentation_metric:   4.544872<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.825352<br />segmentation_metric:   6.624374<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  84.685050<br />segmentation_metric:   4.856419<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.308107<br />segmentation_metric:   5.382084<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.428888<br />segmentation_metric:   5.224344<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  57.426742<br />segmentation_metric:   3.440217<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.105337<br />segmentation_metric:   5.525532<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 115.916740<br />segmentation_metric:   6.159269<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 135.521055<br />segmentation_metric:   4.753498<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 104.749321<br />segmentation_metric:   8.080078<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.898929<br />segmentation_metric:   5.974790<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.060610<br />segmentation_metric:   4.774757<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  67.425433<br />segmentation_metric:   5.149533<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 108.284106<br />segmentation_metric:   8.494774<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.058133<br />segmentation_metric:   5.602151<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.861864<br />segmentation_metric:   5.692537<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  66.529325<br />segmentation_metric:   6.718447<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.108772<br />segmentation_metric:   7.435262<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.700821<br />segmentation_metric:   5.778243<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  35.731573<br />segmentation_metric:   2.355932<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  43.965337<br />segmentation_metric:   3.477419<br />well_name: E09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.337966<br />segmentation_metric:   7.232911<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.530069<br />segmentation_metric:   6.221284<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.522775<br />segmentation_metric:   5.961735<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  56.303712<br />segmentation_metric:   6.300836<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  71.480757<br />segmentation_metric:   5.741036<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 114.161194<br />segmentation_metric:   7.090620<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.268371<br />segmentation_metric:   6.232517<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.306820<br />segmentation_metric:   7.081498<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  76.459744<br />segmentation_metric:   6.488550<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  10.650048<br />segmentation_metric:   3.294118<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.530887<br />segmentation_metric:   4.804924<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 107.185195<br />segmentation_metric:   6.720217<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.331581<br />segmentation_metric:   6.584577<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.572219<br />segmentation_metric:   8.502500<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  66.511614<br />segmentation_metric:   7.087719<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.587282<br />segmentation_metric:   5.450730<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 101.673788<br />segmentation_metric:   6.932735<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 106.236345<br />segmentation_metric:   5.951417<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.018043<br />segmentation_metric:   4.843956<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.803191<br />segmentation_metric:   7.768199<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.230961<br />segmentation_metric:   5.463519<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.188716<br />segmentation_metric:   5.213144<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.571118<br />segmentation_metric:   1.900000<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  55.168534<br />segmentation_metric:   4.145963<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  17.766977<br />segmentation_metric:   3.812500<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 133.811403<br />segmentation_metric:   7.927570<br />well_name: E09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  58.910017<br />segmentation_metric:   1.953596<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  48.943710<br />segmentation_metric:   4.039514<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.572497<br />segmentation_metric:   5.789819<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.821256<br />segmentation_metric:   2.497219<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.775458<br />segmentation_metric:   5.646341<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  75.308434<br />segmentation_metric:   5.979003<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.047810<br />segmentation_metric:   5.226829<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.300822<br />segmentation_metric:   8.076471<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  21.106869<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  61.499267<br />segmentation_metric:   6.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.045771<br />segmentation_metric:   5.810734<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.435225<br />segmentation_metric:   4.428843<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 107.000731<br />segmentation_metric:   6.625287<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  98.476547<br />segmentation_metric:   5.944109<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.773050<br />segmentation_metric:   6.627389<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.365631<br />segmentation_metric:   6.596929<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 107.565618<br />segmentation_metric:   5.876866<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.494438<br />segmentation_metric:   5.077778<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 117.750010<br />segmentation_metric:   6.901487<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 102.999818<br />segmentation_metric:   5.374346<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  82.679937<br />segmentation_metric:   6.446575<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.037211<br />segmentation_metric:   6.666667<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.548949<br />segmentation_metric:   5.602180<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.652278<br />segmentation_metric:   8.447059<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.847148<br />segmentation_metric:   6.072941<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.605655<br />segmentation_metric:   5.673111<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  40.666192<br />segmentation_metric:   3.796117<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  84.311250<br />segmentation_metric:   4.848896<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  48.866134<br />segmentation_metric:   2.074813<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  20.029900<br />segmentation_metric:   1.000000<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.991286<br />segmentation_metric:  10.617470<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  17.310185<br />segmentation_metric:   1.597938<br />well_name: E09 <br>well_pos_x: 4 <br>well_pos_y: 6"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-12.49273495,313.15425595],"y":[1.1,1.1],"text":"yintercept: 1.1","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-12.49273495,313.15425595],"y":[13,13],"text":"yintercept: 13","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":43.1050228310502},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-12.49273495,313.15425595],"tickmode":"array","ticktext":["0","100","200","300"],"tickvals":[-1.77635683940025e-015,100,200,300],"categoryorder":"array","categoryarray":["0","100","200","300"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"Mean_Morphology_Major_Axis_Length","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-9.02290322580645,196.080967741935],"tickmode":"array","ticktext":["0","50","100","150"],"tickvals":[0,50,100,150],"categoryorder":"array","categoryarray":["0","50","100","150"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"segmentation_metric","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"19081c6c7008":{"x":{},"y":{},"text":{},"type":"scatter"},"19081f627ef7":{"yintercept":{}},"190847352867":{"yintercept":{}}},"cur_data":"19081c6c7008","visdat":{"19081c6c7008":["function (y) ","x"],"19081f627ef7":["function (y) ","x"],"190847352867":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-a9d5832e95e9b195fb48">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>