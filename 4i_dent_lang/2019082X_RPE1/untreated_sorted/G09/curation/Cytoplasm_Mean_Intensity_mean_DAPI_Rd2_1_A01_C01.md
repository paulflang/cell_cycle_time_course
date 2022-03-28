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
<div id="htmlwidget-ecebc5ec0a693352fba4" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-ecebc5ec0a693352fba4">{"x":{"data":[{"x":[268939,268940,268941,268942,268943,268944,268945,268946,268947,268949,268950,268951,268952,268953,268954,268956,268957,268958,268959,268960,268961,268962,268963,268964,268965,268966,268967,268968,271448,271450,271451,271452,271453,271454,271455,271456,271457,271459,271460,271461,271462,271463,271464,271465,271467,271468,271470,271471,271472,271473,271474,271476,271477,271478,271479,271480,271481,264702,264704,264705,264706,264707,264709,264710,264711,264712,264713,264714,264715,264716,264717,264718,264719,264721,264722,264723,264724,264725,264726,264727,264728,264730,264731,264732,264733,264734,264737,264738,264739,264740,264741,264742,264743,264744,264745,264746,264747,264748,264749,264750,264752,264753,264755,264756,264757,264758,264759,278454,278456,278457,278458,278459,278460,278461,278462,278463,278465,278466,278467,278468,278469,278470,278471,278472,278473,278474,278475,285775,285776,285777,285778,285779,285780,285781,285782,285783,285784,285785,285786,285787,285788,285789,285790,285791,285792,285793,285794,285798,285803,274776,274777,274778,274779,274781,274782,274784,274785,274786,274787,274789,274790,274791,274792,274793,274795,274796,274797,274799,286183,286184,286185,286188,286189,286190,286191,286193,286194,286195,286197,286199,286200,286202,286203,286204,286206,286207,286208,286209,286211,286212,286213,286216,273084,273087,273088,273089,273090,273092,273093,273094,273095,273096,273097,273098,273099,273100,273101,273103,273106,273107,273109,273110,273111,273112,273114,273115,273116,273117,273118,273119,273120,273121,273122,273123,273124,273127,282201,282203,282204,282205,282206,282207,282208,282209,282210,282211,282212,282214,282215,282216,282217,282218,282219,282220,282221,282223,282224,282228,276762,276763,276764,276765,276766,276767,276768,276769,276770,276771,276772,276773,276774,276775,276776,276777,276778,276779,276780,276781,276782,276786,285912,285913,285915,285916,285917,285918,285919,285920,285921,285922,285923,285924,285925,285926,285927,285928,285929,285930,285931,285932,285933,285934,285936,285939,285940,276394,276395,276397,276398,276399,276400,276401,276402,276404,276405,276406,276407,276408,276409,276410,276411,276412,276413,276414,276415,276416,276417,276418,276419,276420,276422,276423,276424,276425,276426,276427,276428,276431,276434,276435,276436,277357,277358,277359,277360,277361,277362,277363,277364,277365,277367,277368,277369,277370,277371,277372,277373,277374,277375,277376,277378,277379,277380,277382,277383,277384,277385,277386,277387,277389,277390,277392,277394,277395,277397,281118,281120,281121,281122,281124,281125,281126,281127,281128,281130,281131,281133,281134,281135,281136,281138,281139,281142,283781,283784,283785,283787,283788,283789,283790,283791,283792,283793,283794,283795,283796,283798,283799,283800,283801,283803,283804,283805,283806,283807,283808,283809,283810,283811,283813,283814,283815,283816,283817,283818,283820,283821,283823,283824,283831,270443,270445,270446,270447,270448,270450,270451,270452,270453,270454,270455,270456,270457,270458,270460,270461,270462,270463,270464,270465,270466,270467,270468,270469,270470,270472,270473,270474,270476,270477,270478,270479,270480,270482,270483,270484,270485,279149,279151,279152,279153,279154,279156,279157,279159,279160,279161,279162,279163,279164,279165,279166,279167,279168,279169,279170,279171,279172,279173,279174,279175,279176,279178,279179,279181,279182,279184,279185,279186,279187,279188,279190,279191,279192,283624,283625,283626,283627,283628,283629,283632,283633,283635,283636,283637,283638,283639,283641,283642,283643,283644,283645,283646,283647,283648,283649,283650,283651,283652,283653,283654,283655,283656,283658,283659,283665,273549,273550,273552,273553,273554,273555,273557,273558,273559,273560,273561,273562,273564,273565,273566,273567,273568,273569,273570,273571,273572,273573,273575,273577,273578,273580,273583,267877,267878,267879,267880,267881,267882,267883,267884,267886,267888,267889,267890,267891,267892,267893,267894,267895,267896,267897,267899,267901,267902,267903,267904,267905,275966,275969,275970,275971,275972,275973,275975,275976,275977,275978,275979,275980,275981,275982,275983,275984,275985,275986,275987,275988,275989,275990,275991,275992,275993,275994,275995,275999,266219,266220,266221,266222,266223,266224,266225,266226,266229,266230,266231,266232,266233,266236,266237,266238,266239,266240,266241,266242,266243,266244,266246,266247,266248,266249,266250,266251,266252,266253,266254,266255,266256,266257,266258,266261,278419,278420,278421,278422,278423,278425,278426,278427,278429,278430,278432,278433,278434,278435,278436,278437,278438,278440,278441,278442,278443,278444,271360,271367,271368,271370,271371,271372,271373,271374,271375,271376,271377,271379,271380,271381,271382,271383,271385,271386,271388,271389,271390,271391,271392,271393,271394,271395,271396,271397,271398,271400,271401,271402,271403,271404,278319,278320,278321,278322,278323,278324,278325,278326,278327,278328,278329,278330,278331,278332,278334,278335,278336,278337,266835,266836,266837,266838,266839,266840,266842,266843,266845,266846,266848,266851,266852,266853,266854,266855,266856,266857,266858,266860,266861,266864,266865,266952,266953,266954,266956,266957,266958,266959,266961,266962,266964,266965,266966,266968,266969,266970,266971,266972,266974,266975,266977,266978,266979,266980,266981,266982,266983,266984,266985,266987,266990,266991,266992,266995,283270,283271,283272,283273,283274,283275,283277,283278,283279,283280,283281,283282,283284,283285,283286,283287,283288,283289,283290,283291,283292,283293,283294,283295,283296,283297,283298,283299,283301,283302,283303,283304,283305,283306,283307,283308,283310,272460,272461,272462,272463,272464,272465,272466,272468,272469,272470,272471,272472,272473,272474,272475,272477,272478,272479,272480,272481,272482,272484,272486,284299,284300,284301,284302,284303,284304,284305,284306,284307,284308,284309,284310,284311,284312,284313,284314,284315,284316,284317,284318,284319,284321,284323,284324,284326,284327,284328,284329,284330,284331,284333,284336,268125,268128,268129,268130,268131,268132,268133,268135,268136,268137,268138,268139,268140,268141,268144,268145,268146,268148,268150,268151,268152,268153,268154,268156,268157,268158,268159,268161,268162,268163,268164,268167,268168,268169,268170,268171,268172,268173,268174,268175,268178,268179,268180,268185,281995,281996,281997,281998,281999,282001,282002,282003,282004,282005,282006,282007,282009,282011,282012,282013,282015,282017,282018,282019,282021,282023,282024,282025,282026,282028,282031,282032,282033,282034,279348,279350,279351,279352,279353,279355,279357,279358,279359,279360,279361,279362,279363,279364,279366,279367,279368,279369,279371,279372,279373,279377,279379,279381,279382,279383,279387,270690,270691,270692,270693,270696,270697,270698,270699,270700,270701,270702,270704,270705,270706,270707,270708,270710,270711,270712,270713,270714,270715,270716,270718,270721,270722,270723,270724,270725,284702,284703,284704,284705,284706,284707,284708,284709,284710,284711,284713,284714,284716,284717,284718,284719,284721,284722,284723,284724,284728,284732],"y":[null,null,null,null,null,null,null,null,15.856604,43.641786,null,null,null,null,null,null,53.388217,41.275015,null,null,null,null,null,null,null,null,null,null,37.7081,34.08846,53.502753,42.25747,null,null,45.63408,49.276409,null,147.455677,39.42637,35.554536,75.873722,49.843023,39.521814,79.405594,47.580982,32.816097,31.750574,43.296489,null,null,41.413346,37.907143,null,null,null,null,null,null,379.743333,null,157.299048,70.309075,81.833411,59.226178,null,null,69.272131,78.084803,43.657796,null,null,101.152239,181.19756,null,92.225879,75.512631,47.266373,null,38.635172,92.560376,48.180045,90.660966,null,27.124722,192.973459,49.115009,43.981245,null,31.280473,null,null,null,41.172712,80.332803,null,45.498501,null,null,null,45.348485,87.014729,53.573134,47.496698,427.357798,43.414379,50.881593,null,null,29.690727,40.430582,58.827044,39.617275,35.569527,40.621538,104.861576,42.421226,145.59446,34.930026,51.045523,56.880094,48.472188,96.704063,196.596605,null,51.407328,null,null,34.062866,54.534715,37.632568,34.275524,36.164062,32.196347,38.948454,33.920086,39.333555,35.845525,37.033345,38.276197,41.315725,50.973602,32.647436,67.670583,75.255856,106.534722,28.906377,27.826766,176.87988,null,193.880531,67.292135,73.840511,53.159091,110.596774,29.976923,null,49.550048,null,35.943307,34.115419,49.405189,null,null,42.585139,35.418338,30.479901,103.597954,27.237284,41.3045,47.509943,42.786238,44.939759,39.894595,53.618815,null,null,37.296318,67.195977,65.354381,31.542151,null,142.649502,null,60.865267,41.272249,40.476967,28.398559,106.697434,32.540942,37.281372,null,null,34.067555,35.769737,32.462565,50.474066,null,null,33.492483,31.84481,null,67.231051,48.686737,null,null,38.213086,null,31.785796,80.52839,72.066719,87.428571,35.70253,39.429669,46.55938,45.87416,37.292896,56.595269,null,12.055556,null,45.289655,70.627558,57.737294,44.673985,79.810948,48.383377,null,null,null,null,null,null,null,null,null,null,null,53.001455,null,40.848401,null,114.976707,null,38.536655,36.42344,115.652584,38.628354,33.351074,null,null,null,null,null,null,null,null,null,null,null,null,null,142.140965,null,null,35.192731,null,null,182.565649,44.551936,31.844074,31.818037,31.534193,30.943953,35.322457,50.071442,35.157895,42.442918,36.015139,33.560764,35.392143,41.2212,29.043436,30.826917,30.187971,null,32.944844,41.266626,37.402272,36.139436,38.33122,72.539647,34.382866,86.27089,null,70.108348,37.1073,23.438313,38.776721,24.702602,44.991268,16.68761,32.025152,66.079179,41.968323,50.202458,39.354775,31.87399,28.426403,33.557573,40.777021,40.528047,32.760271,24.994616,34.489856,29.559699,29.546675,46.918715,29.906343,34.912595,30.137604,51.941841,41.758751,41.849039,43.080616,46.623681,32.464257,39.482306,29.616438,33.135901,33.928199,71.68314,39.760594,36.947168,32.133492,37.9793,42.466808,46.613415,48.939317,35.519392,32.425086,40.735104,43.392168,39.511679,59.592597,24.190177,99.093458,49.620388,37.156526,64.503354,52.534231,155.923588,52.54561,31.204806,25.530706,26.388005,31.839598,32.848369,28.288462,35.940268,38.12728,32.553147,30.53066,31.943396,34.571252,29.74977,30.68799,null,39.273814,33.950403,60.805708,34.579592,36.593774,42.669736,29.128649,null,40.385549,27.2633,60.798222,43.006187,43.823396,52.569963,27.79465,36.912779,34.009338,null,42.047516,32.133605,38.914013,null,29.120531,33.80136,35.748142,77.097755,30.27773,null,33.916944,60.811843,28.168727,34.324028,31.181705,36.511674,35.030991,45.462805,28.422366,208.024879,51.250892,27.453762,43.493459,37.652556,30.406557,33.997253,44.302872,29.05176,49.201788,192.431889,35.401227,27.932873,38.768824,33.505424,null,39.546302,null,53.874498,42.21939,32.157682,35.123457,34.640963,30.725992,35.259219,32.808435,31.148454,36.132112,33.707258,36.997858,29.470534,36.652679,35.695078,28.523309,31.505098,27.598061,39.178339,34.058958,53.284091,27.22561,41.885724,56.728636,27.979253,27.569227,55.01769,43.039204,23.852532,38.775051,32.829083,31.548134,56.906085,64.994419,31.523815,29.961319,48.182243,28.548067,37.369975,40.355824,36.353451,32.761011,29.696438,29.18363,null,31.707738,79.642151,38.131403,106.844654,43.545839,32.352608,34.030342,36.601501,32.582524,33.385279,33.130996,30.542031,29.266023,33.752286,46.267366,31.995931,27.631929,32.843437,32.045237,39.575248,22.607897,41.322353,52.376227,29.595552,null,27.553888,34.395911,37.528696,45.458754,40.443817,30.388122,36.790015,27.46314,20.732121,34.505303,31.396337,40.009901,64.417748,39.915528,41.698979,33.598779,31.664557,36.652199,31.875069,39.997209,29.66521,33.189045,57.515245,38.030427,36.489708,41.889625,41.658582,33.528644,42.658837,31.141884,36.766954,34.342286,29.430698,41.893915,64.103746,32.477855,81.789818,39.954575,45.279815,40.687432,37.522388,32.871067,68.344569,60.619501,46.612188,31.062242,46.420716,23.631468,27.082581,31.686765,26.27085,30.037083,28.801667,35.78495,29.454335,25.868481,52.476447,39.246442,96.981579,35.475728,100.395683,40.691478,39.933699,43.585308,43.13575,46.070733,null,39.716015,43.665167,43.645771,43.925094,43.573864,43.726836,41.091676,46.039536,39.74252,28.628058,37.203484,174.184681,29.635504,32.418519,39.228974,46.533397,29.485154,37.230565,111.582287,29.68491,123.190021,31.210014,39.831134,35.280961,44.938241,36.572452,37.580332,36.460791,35.257063,50.568323,43.938208,41.224413,30.302995,34.537083,58.140461,25.687204,30.293704,36.783854,39.175667,63.766675,43.961061,35.921854,37.072855,30.049719,33.19403,31.642593,71.089769,35.042029,37.084844,33.63449,36.393691,28.491561,40.067308,28.695927,42.0653,22.729023,39.616451,39.416667,26.381798,35.641015,26.87814,41.852196,31.194671,34.678899,29.198478,31.544802,34.719489,38.635075,34.7869,32.144117,34.365411,29.892009,35.651106,36.402432,34.896214,49.043283,29.46903,32.78118,36.330298,36.739199,32.144514,34.647193,29.166422,40.845506,41.58933,42.384073,44.738497,31.110655,27.571472,30.537745,39.632989,30.222699,33.581466,35.602307,36.938561,31.273749,30.928824,30.722599,32.631795,29.560629,31.908708,39.644729,28.115159,35.678439,55.410494,31.164967,122.676533,37.401107,33.964733,28.776404,170.047941,57.44147,38.380536,38.870147,36.159619,38.816125,31.513746,31.618235,40.944778,33.078089,43.661538,34.810496,35.211222,36.658589,31.656137,31.154836,40.692741,30.384846,29.691201,36.115687,34.725472,37.217905,34.595548,38.270682,35.662146,51.44308,33.458354,29.295658,33.950043,34.054715,60.038542,34.269577,39.42341,34.542873,37.996174,33.69895,33.984361,65.086698,30.50042,28.61067,29.571837,41.807545,34.664915,23.608536,37.211454,155.498475,44.400112,31.817715,36.034067,36.085922,44.072179,37.868486,34.222186,51.640611,28.739293,37.729341,44.100605,34.319432,30.943249,55.834906,45.613031,43.204245,30.349235,39.16296,36.34704,31.553329,35.113711,32.420666,33.960966,88.809524,34.994771,47.287796,37.697974,39.324155,37.500607,41.73819,29.775313,33.0242,31.795304,35.193062,41.789559,62.407982,36.145412,34.379254,74.310345,32.550903,31.941733,42.716256,36.507512,59.189753,54.380158,35.13427,35.3817,46.47191,30.826353,34.804451,42.787709,40.514903,36.241339,66.141095,37.270243,71.276415,102.748284,36.355959,40.600581,41.873282,35.064503,29.480043,32.763337,32.607867,33.861478,38.630901,50.31432,97.247813,37.597757,38.673647,105.560885,126.421739,38.897376,30.194022,37.643282,39.172805,36.872003,26.885338,28.598977,43.141153,27.787752,31.804966,31.493635,36.278194,34.717172,32.226715,39.551504,30.69701,30.184211,36.64455,40.554648,31.257488,34.775855,20.138918,35.230966,107.544554,81.954668,30.537935,37.45405,null,37.636364,43.734649,69.234921,34.135084,30.905495,37.482192,33.777816,60.896552,44.450855,46.313137,30.761194,29.337445,37.271061,26.775774,44.57112,50.493794,46.254366,37.015215,32.935312,70.772727,124.288,36.324974,121.882229,32.982514,182.872373,28.526846,28.917738,28.304708,24.666754,30.862292,27.572062,35.423058,34.344524,24.804204,25.211969,30.971932,36.381723,27.938776,32.629866,32.962351,142.35629,43.259317,25.825497,26.070136,34.481075,45.993256,36.742979,30.983882,35.629184,43.732131,49.441895,39.297743,46.03796,42.335987,46.68667,45.315816,42.193038,16.989445,null,38.108485,43.063743,51.804907,39.379089,37.66327,46.363814,31.501652,31.269644,24.106367,31.461127,227.681818,32.730027,null,35.055824,59.67339,31.202312,33.318317,34.908348,35.038886,49.804852,35.600765,35.396812,36.271619,45.361802,43.554005,51.020172,83.174721,44.186207,48.372732,135.011429,213.6375,null,68.028493,51.16561,40.236769,27.979133,43.654131,31.185621,34.123026,30.66965,34.777265,41.131304,34.054983,29.982312,44.726036,37.59735,33.727454,34.110934,32.626537,34.056659,36.160568,41.487587,32.833747,34.066228,27.708473,29.601993,52.322511,33.648641,65.794794,38.14232,39.844568,41.655328,36.082792,40.692414,39.322178,33.89881,33.982794,34.758116,81.135436,91.473563,33.897025,34.656078,39.150638,31.961704,34.438892,33.314878,37.280068,29.586972,33.735863,34.110396,71.274,37.350948,37.008127,36.558578,54.951314,31.788046,68.203399,42.5827,63.431293,null,null,30.824526,43.042787,44.487288,56.526461,41.762882,39.235925,33.459119,34.623631,52.217523,30.058898,38.273649,30.494772,24.531859,28.823833,33.720437,30.709751,170.11,46.152299,37.14526,37.216372,28.710169,34.008815,null,62.279107,42.63518,29.872361,31.581395,87.944661,95.565315,37.366267,42.875658,40.618462,30.687517,47.355516,25.767639,29.809505,56.697828,30.107068,34.742149,28.493584,30.485558,52.155059,34.148317,32.570454,40.184884,30.586124,27.295217,33.078384,null,null,33.594595],"text":["mapobject_id: 268939<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268940<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268941<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268942<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268943<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268944<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268945<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268946<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268947<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  15.85660","mapobject_id: 268949<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.64179","mapobject_id: 268950<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268951<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268952<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268953<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268954<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268956<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268957<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.38822","mapobject_id: 268958<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.27502","mapobject_id: 268959<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268960<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268961<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268962<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268963<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268964<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268965<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268966<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268967<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268968<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271448<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.70810","mapobject_id: 271450<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.08846","mapobject_id: 271451<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.50275","mapobject_id: 271452<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.25747","mapobject_id: 271453<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271454<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271455<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.63408","mapobject_id: 271456<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.27641","mapobject_id: 271457<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271459<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 147.45568","mapobject_id: 271460<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.42637","mapobject_id: 271461<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.55454","mapobject_id: 271462<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  75.87372","mapobject_id: 271463<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.84302","mapobject_id: 271464<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.52181","mapobject_id: 271465<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  79.40559","mapobject_id: 271467<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.58098","mapobject_id: 271468<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.81610","mapobject_id: 271470<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.75057","mapobject_id: 271471<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.29649","mapobject_id: 271472<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271473<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271474<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.41335","mapobject_id: 271476<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.90714","mapobject_id: 271477<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271478<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271479<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271480<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 271481<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264702<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264704<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 379.74333","mapobject_id: 264705<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264706<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 157.29905","mapobject_id: 264707<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  70.30908","mapobject_id: 264709<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  81.83341","mapobject_id: 264710<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  59.22618","mapobject_id: 264711<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264712<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264713<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  69.27213","mapobject_id: 264714<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  78.08480","mapobject_id: 264715<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.65780","mapobject_id: 264716<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264717<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264718<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 101.15224","mapobject_id: 264719<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 181.19756","mapobject_id: 264721<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264722<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  92.22588","mapobject_id: 264723<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  75.51263","mapobject_id: 264724<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.26637","mapobject_id: 264725<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264726<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.63517","mapobject_id: 264727<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  92.56038","mapobject_id: 264728<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.18004","mapobject_id: 264730<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  90.66097","mapobject_id: 264731<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264732<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.12472","mapobject_id: 264733<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 192.97346","mapobject_id: 264734<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.11501","mapobject_id: 264737<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.98125","mapobject_id: 264738<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264739<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.28047","mapobject_id: 264740<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264741<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264742<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264743<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.17271","mapobject_id: 264744<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  80.33280","mapobject_id: 264745<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264746<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.49850","mapobject_id: 264747<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264748<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264749<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 264750<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.34848","mapobject_id: 264752<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  87.01473","mapobject_id: 264753<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.57313","mapobject_id: 264755<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.49670","mapobject_id: 264756<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 427.35780","mapobject_id: 264757<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.41438","mapobject_id: 264758<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.88159","mapobject_id: 264759<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 278454<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 278456<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.69073","mapobject_id: 278457<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.43058","mapobject_id: 278458<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  58.82704","mapobject_id: 278459<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.61727","mapobject_id: 278460<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.56953","mapobject_id: 278461<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.62154","mapobject_id: 278462<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 104.86158","mapobject_id: 278463<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.42123","mapobject_id: 278465<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 145.59446","mapobject_id: 278466<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.93003","mapobject_id: 278467<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.04552","mapobject_id: 278468<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.88009","mapobject_id: 278469<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.47219","mapobject_id: 278470<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  96.70406","mapobject_id: 278471<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 196.59661","mapobject_id: 278472<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 278473<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.40733","mapobject_id: 278474<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 278475<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 285775<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.06287","mapobject_id: 285776<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  54.53471","mapobject_id: 285777<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.63257","mapobject_id: 285778<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.27552","mapobject_id: 285779<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.16406","mapobject_id: 285780<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.19635","mapobject_id: 285781<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.94845","mapobject_id: 285782<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.92009","mapobject_id: 285783<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.33355","mapobject_id: 285784<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.84553","mapobject_id: 285785<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.03334","mapobject_id: 285786<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.27620","mapobject_id: 285787<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.31573","mapobject_id: 285788<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.97360","mapobject_id: 285789<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.64744","mapobject_id: 285790<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  67.67058","mapobject_id: 285791<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  75.25586","mapobject_id: 285792<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 106.53472","mapobject_id: 285793<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.90638","mapobject_id: 285794<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.82677","mapobject_id: 285798<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 176.87988","mapobject_id: 285803<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 274776<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 193.88053","mapobject_id: 274777<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  67.29214","mapobject_id: 274778<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  73.84051","mapobject_id: 274779<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.15909","mapobject_id: 274781<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 110.59677","mapobject_id: 274782<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.97692","mapobject_id: 274784<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 274785<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.55005","mapobject_id: 274786<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 274787<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.94331","mapobject_id: 274789<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.11542","mapobject_id: 274790<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.40519","mapobject_id: 274791<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 274792<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 274793<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.58514","mapobject_id: 274795<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.41834","mapobject_id: 274796<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.47990","mapobject_id: 274797<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 103.59795","mapobject_id: 274799<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.23728","mapobject_id: 286183<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.30450","mapobject_id: 286184<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.50994","mapobject_id: 286185<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.78624","mapobject_id: 286188<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.93976","mapobject_id: 286189<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.89460","mapobject_id: 286190<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.61881","mapobject_id: 286191<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 286193<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 286194<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.29632","mapobject_id: 286195<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  67.19598","mapobject_id: 286197<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  65.35438","mapobject_id: 286199<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.54215","mapobject_id: 286200<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 286202<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 142.64950","mapobject_id: 286203<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 286204<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.86527","mapobject_id: 286206<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.27225","mapobject_id: 286207<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.47697","mapobject_id: 286208<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.39856","mapobject_id: 286209<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 106.69743","mapobject_id: 286211<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.54094","mapobject_id: 286212<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.28137","mapobject_id: 286213<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 286216<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273084<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.06755","mapobject_id: 273087<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.76974","mapobject_id: 273088<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.46256","mapobject_id: 273089<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.47407","mapobject_id: 273090<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273092<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273093<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.49248","mapobject_id: 273094<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.84481","mapobject_id: 273095<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273096<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  67.23105","mapobject_id: 273097<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.68674","mapobject_id: 273098<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273099<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273100<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.21309","mapobject_id: 273101<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273103<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.78580","mapobject_id: 273106<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  80.52839","mapobject_id: 273107<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  72.06672","mapobject_id: 273109<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  87.42857","mapobject_id: 273110<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.70253","mapobject_id: 273111<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.42967","mapobject_id: 273112<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.55938","mapobject_id: 273114<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.87416","mapobject_id: 273115<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.29290","mapobject_id: 273116<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.59527","mapobject_id: 273117<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273118<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  12.05556","mapobject_id: 273119<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 273120<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.28966","mapobject_id: 273121<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  70.62756","mapobject_id: 273122<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  57.73729","mapobject_id: 273123<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.67399","mapobject_id: 273124<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  79.81095","mapobject_id: 273127<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.38338","mapobject_id: 282201<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282203<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282204<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282205<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282206<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282207<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282208<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282209<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282210<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282211<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282212<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282214<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.00145","mapobject_id: 282215<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282216<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.84840","mapobject_id: 282217<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282218<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 114.97671","mapobject_id: 282219<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 282220<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.53666","mapobject_id: 282221<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.42344","mapobject_id: 282223<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 115.65258","mapobject_id: 282224<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.62835","mapobject_id: 282228<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.35107","mapobject_id: 276762<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276763<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276764<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276765<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276766<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276767<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276768<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276769<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276770<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276771<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276772<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276773<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276774<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276775<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 142.14096","mapobject_id: 276776<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276777<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276778<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.19273","mapobject_id: 276779<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276780<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 276781<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 182.56565","mapobject_id: 276782<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.55194","mapobject_id: 276786<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.84407","mapobject_id: 285912<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.81804","mapobject_id: 285913<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.53419","mapobject_id: 285915<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.94395","mapobject_id: 285916<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.32246","mapobject_id: 285917<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.07144","mapobject_id: 285918<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.15790","mapobject_id: 285919<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.44292","mapobject_id: 285920<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.01514","mapobject_id: 285921<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.56076","mapobject_id: 285922<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.39214","mapobject_id: 285923<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.22120","mapobject_id: 285924<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.04344","mapobject_id: 285925<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.82692","mapobject_id: 285926<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.18797","mapobject_id: 285927<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 285928<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.94484","mapobject_id: 285929<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.26663","mapobject_id: 285930<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.40227","mapobject_id: 285931<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.13944","mapobject_id: 285932<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.33122","mapobject_id: 285933<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  72.53965","mapobject_id: 285934<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.38287","mapobject_id: 285936<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  86.27089","mapobject_id: 285939<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 285940<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  70.10835","mapobject_id: 276394<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.10730","mapobject_id: 276395<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  23.43831","mapobject_id: 276397<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.77672","mapobject_id: 276398<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.70260","mapobject_id: 276399<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.99127","mapobject_id: 276400<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  16.68761","mapobject_id: 276401<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.02515","mapobject_id: 276402<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  66.07918","mapobject_id: 276404<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.96832","mapobject_id: 276405<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.20246","mapobject_id: 276406<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.35477","mapobject_id: 276407<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.87399","mapobject_id: 276408<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.42640","mapobject_id: 276409<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.55757","mapobject_id: 276410<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.77702","mapobject_id: 276411<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.52805","mapobject_id: 276412<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.76027","mapobject_id: 276413<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.99462","mapobject_id: 276414<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.48986","mapobject_id: 276415<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.55970","mapobject_id: 276416<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.54668","mapobject_id: 276417<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.91871","mapobject_id: 276418<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.90634","mapobject_id: 276419<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.91260","mapobject_id: 276420<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.13760","mapobject_id: 276422<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.94184","mapobject_id: 276423<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.75875","mapobject_id: 276424<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.84904","mapobject_id: 276425<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.08062","mapobject_id: 276426<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.62368","mapobject_id: 276427<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.46426","mapobject_id: 276428<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.48231","mapobject_id: 276431<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.61644","mapobject_id: 276434<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.13590","mapobject_id: 276435<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.92820","mapobject_id: 276436<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  71.68314","mapobject_id: 277357<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.76059","mapobject_id: 277358<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.94717","mapobject_id: 277359<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.13349","mapobject_id: 277360<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.97930","mapobject_id: 277361<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.46681","mapobject_id: 277362<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.61342","mapobject_id: 277363<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.93932","mapobject_id: 277364<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.51939","mapobject_id: 277365<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.42509","mapobject_id: 277367<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.73510","mapobject_id: 277368<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.39217","mapobject_id: 277369<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.51168","mapobject_id: 277370<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  59.59260","mapobject_id: 277371<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.19018","mapobject_id: 277372<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  99.09346","mapobject_id: 277373<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.62039","mapobject_id: 277374<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.15653","mapobject_id: 277375<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  64.50335","mapobject_id: 277376<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.53423","mapobject_id: 277378<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 155.92359","mapobject_id: 277379<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.54561","mapobject_id: 277380<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.20481","mapobject_id: 277382<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.53071","mapobject_id: 277383<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.38800","mapobject_id: 277384<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.83960","mapobject_id: 277385<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.84837","mapobject_id: 277386<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.28846","mapobject_id: 277387<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.94027","mapobject_id: 277389<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.12728","mapobject_id: 277390<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.55315","mapobject_id: 277392<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.53066","mapobject_id: 277394<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.94340","mapobject_id: 277395<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.57125","mapobject_id: 277397<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.74977","mapobject_id: 281118<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.68799","mapobject_id: 281120<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 281121<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.27381","mapobject_id: 281122<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.95040","mapobject_id: 281124<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.80571","mapobject_id: 281125<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.57959","mapobject_id: 281126<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.59377","mapobject_id: 281127<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.66974","mapobject_id: 281128<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.12865","mapobject_id: 281130<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 281131<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.38555","mapobject_id: 281133<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.26330","mapobject_id: 281134<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.79822","mapobject_id: 281135<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.00619","mapobject_id: 281136<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.82340","mapobject_id: 281138<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.56996","mapobject_id: 281139<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.79465","mapobject_id: 281142<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.91278","mapobject_id: 283781<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.00934","mapobject_id: 283784<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 283785<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.04752","mapobject_id: 283787<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.13361","mapobject_id: 283788<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.91401","mapobject_id: 283789<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 283790<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.12053","mapobject_id: 283791<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.80136","mapobject_id: 283792<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.74814","mapobject_id: 283793<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  77.09776","mapobject_id: 283794<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.27773","mapobject_id: 283795<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 283796<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.91694","mapobject_id: 283798<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.81184","mapobject_id: 283799<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.16873","mapobject_id: 283800<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.32403","mapobject_id: 283801<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.18171","mapobject_id: 283803<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.51167","mapobject_id: 283804<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.03099","mapobject_id: 283805<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.46281","mapobject_id: 283806<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.42237","mapobject_id: 283807<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 208.02488","mapobject_id: 283808<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.25089","mapobject_id: 283809<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.45376","mapobject_id: 283810<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.49346","mapobject_id: 283811<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.65256","mapobject_id: 283813<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.40656","mapobject_id: 283814<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.99725","mapobject_id: 283815<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.30287","mapobject_id: 283816<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.05176","mapobject_id: 283817<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.20179","mapobject_id: 283818<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 192.43189","mapobject_id: 283820<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.40123","mapobject_id: 283821<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.93287","mapobject_id: 283823<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.76882","mapobject_id: 283824<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.50542","mapobject_id: 283831<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 270443<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.54630","mapobject_id: 270445<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 270446<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.87450","mapobject_id: 270447<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.21939","mapobject_id: 270448<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.15768","mapobject_id: 270450<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.12346","mapobject_id: 270451<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.64096","mapobject_id: 270452<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.72599","mapobject_id: 270453<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.25922","mapobject_id: 270454<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.80844","mapobject_id: 270455<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.14845","mapobject_id: 270456<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.13211","mapobject_id: 270457<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.70726","mapobject_id: 270458<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.99786","mapobject_id: 270460<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.47053","mapobject_id: 270461<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.65268","mapobject_id: 270462<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.69508","mapobject_id: 270463<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.52331","mapobject_id: 270464<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.50510","mapobject_id: 270465<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.59806","mapobject_id: 270466<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.17834","mapobject_id: 270467<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.05896","mapobject_id: 270468<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  53.28409","mapobject_id: 270469<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.22561","mapobject_id: 270470<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.88572","mapobject_id: 270472<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.72864","mapobject_id: 270473<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.97925","mapobject_id: 270474<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.56923","mapobject_id: 270476<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  55.01769","mapobject_id: 270477<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.03920","mapobject_id: 270478<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  23.85253","mapobject_id: 270479<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.77505","mapobject_id: 270480<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.82908","mapobject_id: 270482<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.54813","mapobject_id: 270483<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.90608","mapobject_id: 270484<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  64.99442","mapobject_id: 270485<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.52381","mapobject_id: 279149<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.96132","mapobject_id: 279151<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.18224","mapobject_id: 279152<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.54807","mapobject_id: 279153<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.36997","mapobject_id: 279154<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.35582","mapobject_id: 279156<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.35345","mapobject_id: 279157<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.76101","mapobject_id: 279159<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.69644","mapobject_id: 279160<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.18363","mapobject_id: 279161<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 279162<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.70774","mapobject_id: 279163<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  79.64215","mapobject_id: 279164<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.13140","mapobject_id: 279165<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 106.84465","mapobject_id: 279166<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.54584","mapobject_id: 279167<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.35261","mapobject_id: 279168<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.03034","mapobject_id: 279169<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.60150","mapobject_id: 279170<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.58252","mapobject_id: 279171<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.38528","mapobject_id: 279172<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.13100","mapobject_id: 279173<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.54203","mapobject_id: 279174<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.26602","mapobject_id: 279175<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.75229","mapobject_id: 279176<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.26737","mapobject_id: 279178<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.99593","mapobject_id: 279179<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.63193","mapobject_id: 279181<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.84344","mapobject_id: 279182<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.04524","mapobject_id: 279184<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.57525","mapobject_id: 279185<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  22.60790","mapobject_id: 279186<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.32235","mapobject_id: 279187<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.37623","mapobject_id: 279188<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.59555","mapobject_id: 279190<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 279191<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.55389","mapobject_id: 279192<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.39591","mapobject_id: 283624<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.52870","mapobject_id: 283625<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.45875","mapobject_id: 283626<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.44382","mapobject_id: 283627<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.38812","mapobject_id: 283628<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.79001","mapobject_id: 283629<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.46314","mapobject_id: 283632<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  20.73212","mapobject_id: 283633<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.50530","mapobject_id: 283635<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.39634","mapobject_id: 283636<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.00990","mapobject_id: 283637<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  64.41775","mapobject_id: 283638<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.91553","mapobject_id: 283639<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.69898","mapobject_id: 283641<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.59878","mapobject_id: 283642<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.66456","mapobject_id: 283643<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.65220","mapobject_id: 283644<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.87507","mapobject_id: 283645<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.99721","mapobject_id: 283646<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.66521","mapobject_id: 283647<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.18905","mapobject_id: 283648<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  57.51525","mapobject_id: 283649<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.03043","mapobject_id: 283650<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.48971","mapobject_id: 283651<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.88963","mapobject_id: 283652<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.65858","mapobject_id: 283653<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.52864","mapobject_id: 283654<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.65884","mapobject_id: 283655<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.14188","mapobject_id: 283656<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.76695","mapobject_id: 283658<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.34229","mapobject_id: 283659<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.43070","mapobject_id: 283665<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.89391","mapobject_id: 273549<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  64.10375","mapobject_id: 273550<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.47785","mapobject_id: 273552<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  81.78982","mapobject_id: 273553<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.95457","mapobject_id: 273554<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.27981","mapobject_id: 273555<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.68743","mapobject_id: 273557<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.52239","mapobject_id: 273558<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.87107","mapobject_id: 273559<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  68.34457","mapobject_id: 273560<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.61950","mapobject_id: 273561<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.61219","mapobject_id: 273562<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.06224","mapobject_id: 273564<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.42072","mapobject_id: 273565<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  23.63147","mapobject_id: 273566<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.08258","mapobject_id: 273567<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.68677","mapobject_id: 273568<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.27085","mapobject_id: 273569<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.03708","mapobject_id: 273570<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.80167","mapobject_id: 273571<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.78495","mapobject_id: 273572<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.45434","mapobject_id: 273573<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.86848","mapobject_id: 273575<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.47645","mapobject_id: 273577<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.24644","mapobject_id: 273578<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  96.98158","mapobject_id: 273580<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.47573","mapobject_id: 273583<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 100.39568","mapobject_id: 267877<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.69148","mapobject_id: 267878<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.93370","mapobject_id: 267879<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.58531","mapobject_id: 267880<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.13575","mapobject_id: 267881<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.07073","mapobject_id: 267882<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 267883<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.71601","mapobject_id: 267884<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.66517","mapobject_id: 267886<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.64577","mapobject_id: 267888<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.92509","mapobject_id: 267889<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.57386","mapobject_id: 267890<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.72684","mapobject_id: 267891<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.09168","mapobject_id: 267892<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.03954","mapobject_id: 267893<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.74252","mapobject_id: 267894<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.62806","mapobject_id: 267895<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.20348","mapobject_id: 267896<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 174.18468","mapobject_id: 267897<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.63550","mapobject_id: 267899<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.41852","mapobject_id: 267901<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.22897","mapobject_id: 267902<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.53340","mapobject_id: 267903<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.48515","mapobject_id: 267904<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.23056","mapobject_id: 267905<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 111.58229","mapobject_id: 275966<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.68491","mapobject_id: 275969<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 123.19002","mapobject_id: 275970<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.21001","mapobject_id: 275971<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.83113","mapobject_id: 275972<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.28096","mapobject_id: 275973<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.93824","mapobject_id: 275975<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.57245","mapobject_id: 275976<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.58033","mapobject_id: 275977<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.46079","mapobject_id: 275978<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.25706","mapobject_id: 275979<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.56832","mapobject_id: 275980<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.93821","mapobject_id: 275981<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.22441","mapobject_id: 275982<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.30299","mapobject_id: 275983<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.53708","mapobject_id: 275984<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  58.14046","mapobject_id: 275985<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.68720","mapobject_id: 275986<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.29370","mapobject_id: 275987<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.78385","mapobject_id: 275988<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.17567","mapobject_id: 275989<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  63.76667","mapobject_id: 275990<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.96106","mapobject_id: 275991<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.92185","mapobject_id: 275992<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.07285","mapobject_id: 275993<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.04972","mapobject_id: 275994<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.19403","mapobject_id: 275995<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.64259","mapobject_id: 275999<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  71.08977","mapobject_id: 266219<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.04203","mapobject_id: 266220<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.08484","mapobject_id: 266221<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.63449","mapobject_id: 266222<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.39369","mapobject_id: 266223<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.49156","mapobject_id: 266224<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.06731","mapobject_id: 266225<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.69593","mapobject_id: 266226<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.06530","mapobject_id: 266229<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  22.72902","mapobject_id: 266230<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.61645","mapobject_id: 266231<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.41667","mapobject_id: 266232<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.38180","mapobject_id: 266233<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.64102","mapobject_id: 266236<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.87814","mapobject_id: 266237<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.85220","mapobject_id: 266238<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.19467","mapobject_id: 266239<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.67890","mapobject_id: 266240<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.19848","mapobject_id: 266241<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.54480","mapobject_id: 266242<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.71949","mapobject_id: 266243<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.63508","mapobject_id: 266244<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.78690","mapobject_id: 266246<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.14412","mapobject_id: 266247<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.36541","mapobject_id: 266248<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.89201","mapobject_id: 266249<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.65111","mapobject_id: 266250<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.40243","mapobject_id: 266251<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.89621","mapobject_id: 266252<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.04328","mapobject_id: 266253<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.46903","mapobject_id: 266254<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.78118","mapobject_id: 266255<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.33030","mapobject_id: 266256<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.73920","mapobject_id: 266257<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.14451","mapobject_id: 266258<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.64719","mapobject_id: 266261<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.16642","mapobject_id: 278419<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.84551","mapobject_id: 278420<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.58933","mapobject_id: 278421<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.38407","mapobject_id: 278422<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.73850","mapobject_id: 278423<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.11066","mapobject_id: 278425<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.57147","mapobject_id: 278426<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.53775","mapobject_id: 278427<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.63299","mapobject_id: 278429<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.22270","mapobject_id: 278430<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.58147","mapobject_id: 278432<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.60231","mapobject_id: 278433<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.93856","mapobject_id: 278434<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.27375","mapobject_id: 278435<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.92882","mapobject_id: 278436<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.72260","mapobject_id: 278437<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.63179","mapobject_id: 278438<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.56063","mapobject_id: 278440<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.90871","mapobject_id: 278441<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.64473","mapobject_id: 278442<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.11516","mapobject_id: 278443<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.67844","mapobject_id: 278444<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  55.41049","mapobject_id: 271360<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.16497","mapobject_id: 271367<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 122.67653","mapobject_id: 271368<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.40111","mapobject_id: 271370<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.96473","mapobject_id: 271371<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.77640","mapobject_id: 271372<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 170.04794","mapobject_id: 271373<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  57.44147","mapobject_id: 271374<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.38054","mapobject_id: 271375<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.87015","mapobject_id: 271376<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.15962","mapobject_id: 271377<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.81612","mapobject_id: 271379<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.51375","mapobject_id: 271380<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.61823","mapobject_id: 271381<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.94478","mapobject_id: 271382<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.07809","mapobject_id: 271383<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.66154","mapobject_id: 271385<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.81050","mapobject_id: 271386<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.21122","mapobject_id: 271388<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.65859","mapobject_id: 271389<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.65614","mapobject_id: 271390<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.15484","mapobject_id: 271391<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.69274","mapobject_id: 271392<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.38485","mapobject_id: 271393<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.69120","mapobject_id: 271394<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.11569","mapobject_id: 271395<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.72547","mapobject_id: 271396<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.21791","mapobject_id: 271397<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.59555","mapobject_id: 271398<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.27068","mapobject_id: 271400<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.66215","mapobject_id: 271401<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.44308","mapobject_id: 271402<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.45835","mapobject_id: 271403<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.29566","mapobject_id: 271404<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.95004","mapobject_id: 278319<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.05472","mapobject_id: 278320<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.03854","mapobject_id: 278321<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.26958","mapobject_id: 278322<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.42341","mapobject_id: 278323<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.54287","mapobject_id: 278324<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.99617","mapobject_id: 278325<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.69895","mapobject_id: 278326<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.98436","mapobject_id: 278327<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  65.08670","mapobject_id: 278328<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.50042","mapobject_id: 278329<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.61067","mapobject_id: 278330<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.57184","mapobject_id: 278331<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.80754","mapobject_id: 278332<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.66492","mapobject_id: 278334<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  23.60854","mapobject_id: 278335<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.21145","mapobject_id: 278336<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 155.49848","mapobject_id: 278337<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.40011","mapobject_id: 266835<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.81771","mapobject_id: 266836<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.03407","mapobject_id: 266837<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.08592","mapobject_id: 266838<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.07218","mapobject_id: 266839<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.86849","mapobject_id: 266840<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.22219","mapobject_id: 266842<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.64061","mapobject_id: 266843<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.73929","mapobject_id: 266845<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.72934","mapobject_id: 266846<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.10061","mapobject_id: 266848<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.31943","mapobject_id: 266851<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.94325","mapobject_id: 266852<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  55.83491","mapobject_id: 266853<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.61303","mapobject_id: 266854<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.20425","mapobject_id: 266855<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.34924","mapobject_id: 266856<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.16296","mapobject_id: 266857<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.34704","mapobject_id: 266858<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.55333","mapobject_id: 266860<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.11371","mapobject_id: 266861<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.42067","mapobject_id: 266864<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.96097","mapobject_id: 266865<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  88.80952","mapobject_id: 266952<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.99477","mapobject_id: 266953<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.28780","mapobject_id: 266954<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.69797","mapobject_id: 266956<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.32415","mapobject_id: 266957<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.50061","mapobject_id: 266958<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.73819","mapobject_id: 266959<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.77531","mapobject_id: 266961<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.02420","mapobject_id: 266962<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.79530","mapobject_id: 266964<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.19306","mapobject_id: 266965<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.78956","mapobject_id: 266966<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  62.40798","mapobject_id: 266968<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.14541","mapobject_id: 266969<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.37925","mapobject_id: 266970<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  74.31034","mapobject_id: 266971<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.55090","mapobject_id: 266972<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.94173","mapobject_id: 266974<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.71626","mapobject_id: 266975<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.50751","mapobject_id: 266977<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  59.18975","mapobject_id: 266978<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  54.38016","mapobject_id: 266979<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.13427","mapobject_id: 266980<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.38170","mapobject_id: 266981<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.47191","mapobject_id: 266982<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.82635","mapobject_id: 266983<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.80445","mapobject_id: 266984<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.78771","mapobject_id: 266985<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.51490","mapobject_id: 266987<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.24134","mapobject_id: 266990<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  66.14110","mapobject_id: 266991<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.27024","mapobject_id: 266992<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  71.27642","mapobject_id: 266995<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 102.74828","mapobject_id: 283270<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.35596","mapobject_id: 283271<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.60058","mapobject_id: 283272<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.87328","mapobject_id: 283273<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.06450","mapobject_id: 283274<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.48004","mapobject_id: 283275<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.76334","mapobject_id: 283277<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.60787","mapobject_id: 283278<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.86148","mapobject_id: 283279<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.63090","mapobject_id: 283280<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.31432","mapobject_id: 283281<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  97.24781","mapobject_id: 283282<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.59776","mapobject_id: 283284<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.67365","mapobject_id: 283285<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 105.56088","mapobject_id: 283286<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 126.42174","mapobject_id: 283287<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.89738","mapobject_id: 283288<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.19402","mapobject_id: 283289<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.64328","mapobject_id: 283290<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.17280","mapobject_id: 283291<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.87200","mapobject_id: 283292<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.88534","mapobject_id: 283293<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.59898","mapobject_id: 283294<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.14115","mapobject_id: 283295<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.78775","mapobject_id: 283296<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.80497","mapobject_id: 283297<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.49364","mapobject_id: 283298<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.27819","mapobject_id: 283299<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.71717","mapobject_id: 283301<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.22671","mapobject_id: 283302<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.55150","mapobject_id: 283303<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.69701","mapobject_id: 283304<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.18421","mapobject_id: 283305<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.64455","mapobject_id: 283306<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.55465","mapobject_id: 283307<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.25749","mapobject_id: 283308<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.77585","mapobject_id: 283310<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  20.13892","mapobject_id: 272460<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.23097","mapobject_id: 272461<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 107.54455","mapobject_id: 272462<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  81.95467","mapobject_id: 272463<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.53794","mapobject_id: 272464<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.45405","mapobject_id: 272465<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 272466<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.63636","mapobject_id: 272468<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.73465","mapobject_id: 272469<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  69.23492","mapobject_id: 272470<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.13508","mapobject_id: 272471<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.90549","mapobject_id: 272472<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.48219","mapobject_id: 272473<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.77782","mapobject_id: 272474<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  60.89655","mapobject_id: 272475<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.45085","mapobject_id: 272477<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.31314","mapobject_id: 272478<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.76119","mapobject_id: 272479<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.33744","mapobject_id: 272480<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.27106","mapobject_id: 272481<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.77577","mapobject_id: 272482<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.57112","mapobject_id: 272484<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  50.49379","mapobject_id: 272486<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.25437","mapobject_id: 284299<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.01521","mapobject_id: 284300<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.93531","mapobject_id: 284301<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  70.77273","mapobject_id: 284302<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 124.28800","mapobject_id: 284303<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.32497","mapobject_id: 284304<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 121.88223","mapobject_id: 284305<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.98251","mapobject_id: 284306<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 182.87237","mapobject_id: 284307<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.52685","mapobject_id: 284308<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.91774","mapobject_id: 284309<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.30471","mapobject_id: 284310<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.66675","mapobject_id: 284311<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.86229","mapobject_id: 284312<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.57206","mapobject_id: 284313<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.42306","mapobject_id: 284314<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.34452","mapobject_id: 284315<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.80420","mapobject_id: 284316<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.21197","mapobject_id: 284317<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.97193","mapobject_id: 284318<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.38172","mapobject_id: 284319<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.93878","mapobject_id: 284321<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.62987","mapobject_id: 284323<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.96235","mapobject_id: 284324<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 142.35629","mapobject_id: 284326<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.25932","mapobject_id: 284327<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.82550","mapobject_id: 284328<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  26.07014","mapobject_id: 284329<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.48107","mapobject_id: 284330<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.99326","mapobject_id: 284331<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.74298","mapobject_id: 284333<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.98388","mapobject_id: 284336<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.62918","mapobject_id: 268125<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.73213","mapobject_id: 268128<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.44190","mapobject_id: 268129<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.29774","mapobject_id: 268130<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.03796","mapobject_id: 268131<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.33599","mapobject_id: 268132<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.68667","mapobject_id: 268133<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.31582","mapobject_id: 268135<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.19304","mapobject_id: 268136<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  16.98944","mapobject_id: 268137<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268138<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.10849","mapobject_id: 268139<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.06374","mapobject_id: 268140<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.80491","mapobject_id: 268141<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.37909","mapobject_id: 268144<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.66327","mapobject_id: 268145<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.36381","mapobject_id: 268146<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.50165","mapobject_id: 268148<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.26964","mapobject_id: 268150<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.10637","mapobject_id: 268151<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.46113","mapobject_id: 268152<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 227.68182","mapobject_id: 268153<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.73003","mapobject_id: 268154<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268156<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.05582","mapobject_id: 268157<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  59.67339","mapobject_id: 268158<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.20231","mapobject_id: 268159<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.31832","mapobject_id: 268161<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.90835","mapobject_id: 268162<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.03889","mapobject_id: 268163<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  49.80485","mapobject_id: 268164<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.60077","mapobject_id: 268167<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  35.39681","mapobject_id: 268168<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.27162","mapobject_id: 268169<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  45.36180","mapobject_id: 268170<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.55400","mapobject_id: 268171<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.02017","mapobject_id: 268172<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  83.17472","mapobject_id: 268173<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.18621","mapobject_id: 268174<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  48.37273","mapobject_id: 268175<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 135.01143","mapobject_id: 268178<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 213.63750","mapobject_id: 268179<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 268180<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  68.02849","mapobject_id: 268185<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  51.16561","mapobject_id: 281995<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.23677","mapobject_id: 281996<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.97913","mapobject_id: 281997<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.65413","mapobject_id: 281998<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.18562","mapobject_id: 281999<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.12303","mapobject_id: 282001<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.66965","mapobject_id: 282002<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.77726","mapobject_id: 282003<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.13130","mapobject_id: 282004<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.05498","mapobject_id: 282005<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.98231","mapobject_id: 282006<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.72604","mapobject_id: 282007<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.59735","mapobject_id: 282009<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.72745","mapobject_id: 282011<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.11093","mapobject_id: 282012<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.62654","mapobject_id: 282013<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.05666","mapobject_id: 282015<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.16057","mapobject_id: 282017<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.48759","mapobject_id: 282018<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.83375","mapobject_id: 282019<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.06623","mapobject_id: 282021<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.70847","mapobject_id: 282023<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.60199","mapobject_id: 282024<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.32251","mapobject_id: 282025<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.64864","mapobject_id: 282026<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  65.79479","mapobject_id: 282028<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.14232","mapobject_id: 282031<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.84457","mapobject_id: 282032<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.65533","mapobject_id: 282033<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.08279","mapobject_id: 282034<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.69241","mapobject_id: 279348<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.32218","mapobject_id: 279350<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.89881","mapobject_id: 279351<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.98279","mapobject_id: 279352<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.75812","mapobject_id: 279353<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  81.13544","mapobject_id: 279355<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  91.47356","mapobject_id: 279357<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.89702","mapobject_id: 279358<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.65608","mapobject_id: 279359<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.15064","mapobject_id: 279360<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.96170","mapobject_id: 279361<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.43889","mapobject_id: 279362<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.31488","mapobject_id: 279363<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.28007","mapobject_id: 279364<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.58697","mapobject_id: 279366<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.73586","mapobject_id: 279367<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.11040","mapobject_id: 279368<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  71.27400","mapobject_id: 279369<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.35095","mapobject_id: 279371<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.00813","mapobject_id: 279372<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  36.55858","mapobject_id: 279373<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  54.95131","mapobject_id: 279377<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.78805","mapobject_id: 279379<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  68.20340","mapobject_id: 279381<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.58270","mapobject_id: 279382<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  63.43129","mapobject_id: 279383<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 279387<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 270690<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.82453","mapobject_id: 270691<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  43.04279","mapobject_id: 270692<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  44.48729","mapobject_id: 270693<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.52646","mapobject_id: 270696<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  41.76288","mapobject_id: 270697<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  39.23593","mapobject_id: 270698<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.45912","mapobject_id: 270699<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.62363","mapobject_id: 270700<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.21752","mapobject_id: 270701<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.05890","mapobject_id: 270702<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  38.27365","mapobject_id: 270704<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.49477","mapobject_id: 270705<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  24.53186","mapobject_id: 270706<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.82383","mapobject_id: 270707<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.72044","mapobject_id: 270708<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.70975","mapobject_id: 270710<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01: 170.11000","mapobject_id: 270711<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  46.15230","mapobject_id: 270712<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.14526","mapobject_id: 270713<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.21637","mapobject_id: 270714<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.71017","mapobject_id: 270715<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.00881","mapobject_id: 270716<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 270718<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  62.27911","mapobject_id: 270721<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.63518","mapobject_id: 270722<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.87236","mapobject_id: 270723<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  31.58140","mapobject_id: 270724<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  87.94466","mapobject_id: 270725<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  95.56531","mapobject_id: 284702<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  37.36627","mapobject_id: 284703<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  42.87566","mapobject_id: 284704<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.61846","mapobject_id: 284705<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.68752","mapobject_id: 284706<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  47.35552","mapobject_id: 284707<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  25.76764","mapobject_id: 284708<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  29.80951","mapobject_id: 284709<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  56.69783","mapobject_id: 284710<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.10707","mapobject_id: 284711<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.74215","mapobject_id: 284713<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  28.49358","mapobject_id: 284714<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.48556","mapobject_id: 284716<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  52.15506","mapobject_id: 284717<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  34.14832","mapobject_id: 284718<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  32.57045","mapobject_id: 284719<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  40.18488","mapobject_id: 284721<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  30.58612","mapobject_id: 284722<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  27.29522","mapobject_id: 284723<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.07838","mapobject_id: 284724<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 284728<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:        NA","mapobject_id: 284732<br />Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01:  33.59459"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[263626.3,287291.7],"y":[95,95],"text":"yintercept: 95","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":43.1050228310502},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[263626.3,287291.7],"tickmode":"array","ticktext":["265000","270000","275000","280000","285000"],"tickvals":[265000,270000,275000,280000,285000],"categoryorder":"array","categoryarray":["265000","270000","275000","280000","285000"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"mapobject_id","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-8.7095561,448.1229101],"tickmode":"array","ticktext":["0","100","200","300","400"],"tickvals":[0,100,200,300,400],"categoryorder":"array","categoryarray":["0","100","200","300","400"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"Cytoplasm_Mean_Intensity_mean_DAPI_Rd2_1_A01_C01","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"19082b0857c0":{"x":{},"y":{},"type":"scatter"},"19087f225871":{"yintercept":{}}},"cur_data":"19082b0857c0","visdat":{"19082b0857c0":["function (y) ","x"],"19087f225871":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-ecebc5ec0a693352fba4">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>