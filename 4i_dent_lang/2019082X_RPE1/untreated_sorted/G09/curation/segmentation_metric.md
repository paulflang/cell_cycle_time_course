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
<div id="htmlwidget-51e92edd0a4befec6a24" class="plotly html-widget" style="width:100%;height:400px;">

</div>
</div>
<script type="application/json" data-for="htmlwidget-51e92edd0a4befec6a24">{"x":{"data":[{"x":[30.588326,21.643376,24.707582,25.916786,29.110442,29.969564,31.887628,30.574978,44.814169,102.31982,26.901107,22.001246,29.906977,36.977386,31.536765,24.788583,128.045299,62.307653,27.490101,25.153857,31.972604,31.574348,30.174627,30.819527,32.16541,26.319083,25.630328,30.374171,94.805598,85.146587,86.323846,84.293901,14.634773,23.62108,100.002066,143.40497,14.708782,73.386219,71.585033,119.221478,50.942189,82.038942,129.731934,41.311222,74.474414,92.200378,57.498064,75.895891,33.734704,36.955449,58.135434,25.885843,32.578014,17.69789,26.647709,30.840836,26.336614,19.536252,28.683599,32.521517,30.327123,152.604953,65.826307,95.661206,22.654649,36.59617,37.422177,65.925947,135.94748,29.295462,27.291154,78.163577,82.740121,30.802741,54.953894,83.357727,151.077127,35.866594,96.485016,84.859545,74.100922,48.19803,24.12628,37.74457,112.097152,90.072003,66.469226,25.806956,69.032152,28.847621,32.736577,25.896118,71.600166,70.95004,28.088653,117.727208,46.470373,31.425151,20.434825,61.054576,106.738445,69.09724,53.03942,32.438375,48.416001,86.407747,28.45257,34.256512,94.119105,92.792455,58.345869,94.955127,124.976135,74.812966,111.836445,101.653736,85.390281,166.963783,84.050455,73.75357,80.481993,98.920918,84.795959,34.703614,101.318272,26.481593,28.036374,104.856895,79.748389,65.641658,108.03235,91.681945,74.611753,83.814328,100.180625,88.353412,81.781461,130.704193,52.762311,59.873617,73.621581,69.465206,92.077983,41.59814,60.728222,91.370995,111.8517,58.356643,30.638232,27.982896,168.015317,62.995803,92.45298,94.08612,33.921205,9.94635,72.00225,19.856721,89.556526,69.278903,47.538694,25.946337,23.574007,51.032326,98.539513,94.690364,92.581007,95.160343,71.737418,96.849551,108.381632,72.949334,64.446349,105.374021,31.57276,36.716137,94.63198,59.207506,99.906576,81.024119,23.356746,80.789051,34.279161,68.046277,62.127543,79.341901,88.23141,91.18708,66.495936,68.469787,37.232914,9.404353,76.280727,83.432389,76.877431,80.038211,26.005039,29.665674,88.153245,112.314343,28.952014,46.243902,56.724711,26.866187,37.777766,96.250682,33.38673,103.662256,120.771589,75.832512,35.340287,129.048063,85.080339,57.698677,87.35219,74.609388,58.741101,38.399508,36.240981,23.23226,71.585586,72.921128,153.59437,67.149965,132.630752,87.766785,27.269932,33.646157,30.750029,31.493015,35.507578,27.76556,31.077138,35.5612,28.431179,29.557058,28.893915,82.365836,33.468598,82.592224,28.120758,50.956435,26.103548,77.20819,71.113022,85.905974,91.97495,133.89834,27.354851,25.49671,28.144674,27.466306,37.196526,30.942806,30.120521,33.944705,27.871443,28.558458,33.075611,32.74451,26.083181,112.877588,33.903994,35.940732,96.133831,25.455033,24.632478,122.912887,61.2486,76.012487,89.690261,126.017169,123.519828,81.972545,87.784196,120.945575,79.739442,110.802793,119.095633,84.183089,123.604645,168.697653,76.630401,100.13242,23.624063,86.922235,81.495612,83.402438,88.147527,101.515107,74.341078,64.2318,33.479204,23.239425,36.156746,67.563048,98.672538,89.893002,71.134644,96.374472,72.724629,104.209725,105.907744,70.832357,241.846407,67.992722,94.505892,165.802892,77.773472,86.265222,65.841455,67.780544,95.519113,93.462231,97.06463,72.267463,142.639458,102.918789,126.173407,81.441944,85.224205,77.226976,78.064978,99.233142,69.991934,84.478385,102.600233,64.429074,142.744876,70.203703,43.081842,126.929533,65.196109,86.988979,67.386309,74.287311,91.672812,105.198002,85.634365,80.082556,89.350163,126.078208,97.351506,94.16776,104.521379,27.739471,152.817186,98.821633,35.112418,73.774951,38.315528,98.241647,116.14454,105.08535,164.640221,84.466857,86.618908,62.822444,75.103797,83.927732,112.728716,97.480952,100.440155,85.83126,75.797709,111.670437,30.304676,93.785267,80.563026,100.220923,99.113199,105.253532,82.109375,106.85289,36.80917,57.563533,107.648812,106.373388,84.899841,92.763767,56.737854,168.386685,78.060021,88.978876,33.547872,103.563328,82.335883,121.923773,32.453618,77.802765,84.145914,97.88406,61.360728,77.72235,27.99785,132.359472,87.593637,100.085051,131.173938,105.415181,96.023571,61.341317,75.65008,91.551475,63.657219,84.782278,70.495974,128.898862,64.8442,86.646628,98.448155,91.271707,87.1677,49.229216,82.915077,68.775514,106.961561,55.815653,100.541014,32.236744,106.032881,35.979954,164.674521,87.921956,86.797522,94.004074,76.497863,93.160829,132.453308,97.713305,94.237616,66.094807,91.443231,88.345516,109.641144,90.574627,71.592962,113.324782,104.296069,96.686351,111.964131,109.367137,52.263401,125.812548,111.314533,45.871978,172.125261,112.018144,83.590785,96.820393,93.539689,97.50049,88.181412,73.733983,63.863815,78.508493,57.43123,104.204507,88.996149,114.861934,76.018761,81.915263,91.232281,81.267933,66.856435,155.811055,36.108105,78.215236,47.511415,63.546899,24.510129,80.857738,90.747922,106.990641,68.209916,100.336084,66.538501,65.28447,66.056613,89.319439,81.152583,49.152231,100.180443,98.869313,63.492467,74.581777,76.689856,74.223872,108.205993,113.467444,90.823598,9.147286,82.852354,65.401114,79.9773,117.035279,66.039949,105.147423,102.864677,92.159493,109.246635,75.225335,108.450795,67.177032,96.312578,53.025302,116.37028,93.037452,86.747938,92.14267,107.754916,57.051071,90.452265,134.861579,27.292427,99.489098,97.719166,46.862213,86.052699,119.895164,82.60221,170.193993,71.045385,91.830096,119.717545,9.345983,41.878714,107.922809,37.082303,62.641793,114.866851,67.741089,66.258469,83.555295,60.522137,73.973158,95.725976,86.733828,75.238636,141.993301,66.770472,88.69629,138.017318,80.589728,87.241101,74.148107,74.386873,123.523098,53.96182,86.379323,35.690264,71.874942,25.20788,77.944527,65.175007,52.859581,71.927387,77.8079,6.353278,86.239804,70.603975,83.807893,82.161458,111.222661,91.180372,59.471799,102.950314,86.336996,75.076154,87.893789,67.524191,87.051273,112.843434,96.828838,87.142596,100.570149,70.753211,56.845147,78.301792,65.758333,91.983185,128.34606,86.591201,110.336455,71.480266,66.644724,60.637606,112.948904,94.635382,87.28083,90.971911,177.024643,106.921014,77.093068,149.70966,123.92906,72.429086,89.747572,92.951007,81.003706,74.813521,73.384499,102.201773,85.52829,69.116427,48.387289,76.532291,77.3837,51.154286,101.25837,127.243733,103.601421,139.22868,59.759593,103.211894,55.161175,94.430422,107.128742,91.961377,114.685855,48.656574,120.689951,133.95639,137.264449,74.642808,151.060097,52.298476,66.568266,103.297928,244.997092,106.436077,83.980523,116.20387,130.313188,108.71318,87.35267,109.909689,83.833489,102.024165,136.061306,109.784672,80.327953,65.795161,63.969831,82.026154,67.80065,108.448152,99.103956,246.65797,68.535259,116.410028,75.862494,61.852985,100.921154,68.139218,63.872313,95.381896,81.930455,101.652226,57.23187,78.637393,76.916809,83.705827,41.652629,123.36405,31.307426,75.071527,81.626041,78.355936,30.129776,60.819582,71.516064,108.745071,141.55092,87.729898,94.298696,102.246499,88.824738,102.857658,99.716283,101.711278,101.051498,100.490774,94.648943,103.440594,69.614595,67.332401,76.868987,76.453636,81.85029,147.601862,81.898409,59.499128,143.388948,72.315163,65.829585,70.100144,101.45051,55.452122,74.563703,126.040551,66.396849,122.45204,70.040464,73.198512,81.676683,71.434923,81.650717,77.220749,83.486783,93.942684,57.606863,96.637411,128.713304,120.141567,101.867147,171.442979,108.328109,74.166577,139.137327,74.448456,87.387825,92.325704,136.514046,118.919374,124.006805,77.602822,81.021322,53.60252,68.594523,171.108432,72.58992,74.428848,173.502456,77.291648,78.246416,114.07613,71.484524,26.743697,74.879231,92.248949,110.088103,86.393464,109.711899,83.795084,131.95902,105.461968,161.980344,127.912643,98.811524,44.129145,88.860524,105.637402,45.968509,98.653172,124.325588,69.027367,137.518695,41.945602,118.710405,70.030248,118.068847,59.556614,116.558922,95.142124,51.566033,62.253965,98.428209,49.385392,78.578054,47.918959,44.430616,60.260547,111.021418,100.057662,80.421729,75.63782,132.991968,87.255592,87.742502,62.850066,76.916143,35.774611,91.913929,83.97484,43.57011,38.143836,91.694486,123.511574,67.141721,103.077461,102.251419,111.493164,105.521825,53.976544,85.855901,116.558076,77.667154,158.79267,63.150014,107.897673,53.551715,116.452968,31.387968,113.614942,86.719539,63.973925,95.354109,84.207764,64.852515,22.995306,50.893986,109.241151,96.383914,4.472136,38.94312,57.429343,48.82137,79.471455,117.447613,104.64782,87.037685,44.897054,59.762215,86.864106,82.465823,180.142301,111.450592,109.834105,85.207678,89.722002,77.551622,90.383366,84.298515,54.146353,23.080442,76.031116,21.265981,78.638582,69.720033,48.198661,82.564659,109.968638,92.874362,81.840329,85.243185,82.618617,67.364873,91.373056,80.683931,87.639631,105.791391,85.339458,111.111034,62.964544,55.746632,108.961315,128.797123,82.584384,76.217201,154.377421,60.528243,84.055965,60.799704,79.095484,71.197137,86.483229,94.313272,124.700798,105.250964,60.910229,71.705365,114.920358,5.437579,68.927319,73.190718,92.469432,51.175747,79.69612,140.476332,100.670527,105.932239,142.265348,86.417594,18.256201,138.290109,6.32461,121.20639,68.83399,81.118813,96.371048,95.488487,69.75768,137.180277,83.295956,85.994555,87.079524,87.903057,58.036477,105.948453,42.007676,51.582408,54.38823,25.167547,16.249092,9.395695,77.012834,12.406573,96.433001,113.417097,134.363653,93.533475,73.157457,90.772813,144.378709,195.32765,88.880003,118.694761,58.218516,99.5206,89.392921,96.988717,94.531465,117.443473,68.930337,97.117019,122.304928,93.967158,89.619177,72.152217,62.00416,84.333895,56.89674,81.009604,53.230712,134.894619,74.028914,40.348587,103.100903,94.500617,157.435203,89.049724,42.693166,43.883568,77.244072,59.781352,80.859164,78.82889,95.985768,96.059027,103.39642,105.47569,97.768303,78.772768,84.995395,136.03041,247.175591,99.644206,83.290124,64.849342,85.988925,88.829635,65.442878,15.792389,16.990421,93.352977,81.565576,63.365458,66.234337,93.257939,88.416598,35.43612,65.098763,83.619572,106.999432,64.222338,124.212862,129.466099,96.709357,148.691444,88.283077,26.043506,52.740683,77.033515,93.43231,109.730126,100.29975,27.17202,93.764211,125.471581,95.232011,72.162102,74.339494,34.525826,87.279325,81.871533,109.216776,142.447656,74.93361,77.215704,95.495918,74.968993,160.397344,83.152007,117.713725,88.890239,136.960985,57.479043,123.885451,70.270647,74.983942,108.428547,121.62274,30.368613,25.091238,17.208258],"y":[1,1,1,1,1,1,1,1,1.45689655172414,9.00797872340426,1,1,1,1,1.00735294117647,1,4.28178694158076,6.29102167182663,1,1,1,1,1,1,1,1,1,1,5.84036144578313,6.17169373549884,4.01490066225166,4.98227848101266,1,1,4.73559322033898,4.53834808259587,1,4.34608985024958,4.19489981785064,4.60505836575875,3.35037878787879,3.15921288014311,12.9872122762148,1.40952380952381,4.86666666666667,6.15862068965517,5.70967741935484,6.08351648351648,1,1,4.17191977077364,1.99342105263158,1,1,1,1,1,1,3.3671875,1,2.61111111111111,6.49342105263158,4.92427184466019,3.80869565217391,1,1,3.43426294820717,6.92537313432836,4.66476624857469,1,1,2.5952380952381,6.0521327014218,1,3.125,7.7625,6.00287356321839,1,4.65619834710744,3.63544668587896,3.65384615384615,2.05679513184584,1,2.36873156342183,6.98974358974359,3.38081395348837,3.22682119205298,1,5.78015564202335,1.06068601583113,1,1,4.41257367387033,4.58174904942966,1,6.44208037825059,1,1,1,4.08411214953271,6.05061082024433,3.74395161290323,5.45424836601307,1.2845953002611,3.33734939759036,4.06102362204724,1,1,6.35600578871201,5.00562851782364,2.26657824933687,5.96316758747698,4.09063444108761,4.63578274760383,6.33580705009276,5.41164241164241,5.09302325581395,8.29179810725552,5.39669421487603,4.2109955423477,5.43692870201097,6.09765625,3.39728682170543,1,4.57969151670951,1,1,9.46137339055794,6.93318965517241,7.79320113314448,6.83116883116883,7.84189723320158,7.27207637231504,5.83116883116883,7.17880794701987,4.04743083003953,8.16558441558442,8.79915730337079,5.66265060240964,5.97859327217125,6.33884297520661,6.59523809523809,6.90133333333333,1.55823293172691,5.87384615384615,8.13801452784504,9.03350970017637,3.60835509138381,1,2.65693430656934,9.43827160493827,4,5.46543778801843,7.19578947368421,1.3983286908078,1,4.65034965034965,1,5.07768924302789,5.38996138996139,3.20466321243523,1,1,4.34715025906736,4.87966804979253,6.24584103512015,6.92540792540793,5.35305719921105,5.13924050632911,5.21601489757914,7.31893265565438,7.75254237288136,6.52507374631268,3.86948176583493,1,1,7.05308219178082,6.19505494505495,7.02623906705539,9.07826086956522,1,8.93233082706767,1,4.88255033557047,5.75675675675676,6.36501901140684,5.8042328042328,86.4242424242424,6.61473087818697,4.08809523809524,1,1,7.43991853360489,6.75840336134454,6.18444444444444,5.21397379912664,1,1,7.05636363636364,9.77090909090909,1,1.58235294117647,7.19095477386935,1,1,5.58908507223114,1,11.1480769230769,4.53988326848249,3.68972746331237,1.26075949367089,8.35148514851485,4.54441260744986,4.32,5.11471321695761,6.69503546099291,3.40978593272171,1,1.05346820809249,1,4.15398550724638,5.15631691648822,8.33388429752066,4.92307692307692,9.73895582329317,4.8969696969697,1,1,1,1,1,1,1,1,1,1,1,3.18895966029724,1,5.46739130434783,1,2.19187358916479,1,4.25091575091575,5.64596273291925,6.70327102803738,4.87258687258687,6.61012311901505,1,1,1,1,1,1,1,1,1,1,1,1,1,7.17059891107078,1,1,6.34050880626223,1,1,3.82464929859719,5.63532763532764,7.08219178082192,9.73611111111111,7.51277372262774,9.84123222748815,6.24120603015075,6.08591065292096,7.13216957605985,5.94613583138173,5.24033613445378,10.9337748344371,7.08149779735683,6.72332730560579,8.50635208711434,8.30940594059406,5.39904988123515,1,6.11029411764706,5.50276243093923,6.61477572559367,5.63300492610837,7.03529411764706,9.21899736147757,5.79545454545455,2.49152542372881,1,2.82792207792208,6.73314606741573,5.42439862542955,8.56332703213611,5.35472370766488,7.90346083788707,5.12115732368897,7.2046875,5.328611898017,4.41176470588235,7.65610859728507,5.75764705882353,8.11389521640091,12.2390852390852,7.34293193717278,6.91457286432161,4.80903490759754,6.55309734513274,5.60178970917226,5.72908366533865,7.17218543046358,4.9375,5.90909090909091,6.07286432160804,6.47097844112769,7.74936061381074,4.8051391862955,5.13698630136986,5.66524520255864,6.08956916099773,5.03499079189687,6.12676056338028,6.53154574132492,4.54713493530499,5.59567387687188,3.63800904977376,3.17350157728707,6.74728682170543,5.59170305676856,7.40347071583514,5.53348214285714,5.54580152671756,6.0840197693575,6.95724907063197,5.39267886855241,7.48106904231626,5.65402843601896,5.10840438489647,16.9929577464789,4.32926829268293,6.68190476190476,2.21468926553672,9.61013986013986,5.0828025477707,1.17344753747323,2.30508474576271,1.76010101010101,4.90488431876607,11.0220750551876,7.62955465587045,6.29969879518072,7.45075757575758,6.54661016949153,5.76315789473684,5.48307692307692,5.67234848484848,6.86243386243386,6.35606060606061,5.86644407345576,8.84745762711864,8.89903846153846,7.07590132827325,1,6.24784482758621,7.10852713178295,8.14518760195759,6.50974930362117,6.52742616033755,7.44761904761905,7.87224669603524,1,4.51162790697674,9.95814977973568,3.83040935672515,4.43650793650794,4.81995661605206,4.97428571428571,7.95354523227384,4.59223300970874,5.28347826086957,1,5.6505016722408,5.422,4.69603524229075,1,5.8887171561051,5.35121951219512,4.91609589041096,4.81491712707182,4.52671755725191,1,7.43850267379679,3.80907877169559,5.97720797720798,5.13555555555556,5.14455782312925,5.01732283464567,4.09044368600683,3.87412587412587,7.5045045045045,3.35598705501618,3.83477011494253,7.78769230769231,5.4249547920434,6.58282208588957,7.43799472295515,4.98540145985401,3.44585987261146,7.20971867007673,2.72087912087912,3.70292887029289,6.67898383371825,6.16205533596838,3.87686567164179,6.45308310991957,1,6.29936305732484,1,5.76555023923445,4.10714285714286,6.72406639004149,6.61346153846154,5.54693140794224,6.10795454545455,7.74630541871921,5.00903225806452,7.4604743083004,3.9404990403071,5.74809160305344,5.10284463894967,6.77992957746479,6.65909090909091,5.66480446927374,9.83090909090909,6.23770491803279,7.22549019607843,6.49478390461997,5.74961360123648,3.60526315789474,8.42887931034483,5.17598908594816,2.95029239766082,8.3048128342246,9.26293103448276,4.66287878787879,4.90322580645161,5.08349146110057,8.31670822942643,6.01899827288428,5.04473684210526,3.65263157894737,5.86046511627907,4.07575757575758,9.03758169934641,6.6790450928382,7.10984848484848,5.36222910216718,5.45325779036827,7.08731466227348,5.08485856905158,5.04904632152589,10.8214747736093,1.06818181818182,5.36883116883117,6.84615384615385,5.15740740740741,2,5.05915492957747,4.79613733905579,8.28012519561815,5.69259259259259,6.19060773480663,6.41208791208791,5.45135135135135,6.22788203753351,7.73672055427252,5.70040485829959,3.58222222222222,7.28683693516699,7.00665557404326,5.49148936170213,8.26666666666667,6.21797752808989,6.5,10.9387755102041,6.21757322175732,11.311170212766,1,8.26237623762376,6.68073878627968,5.54420432220039,6.28350515463918,5.90581717451524,6.46180555555556,6.68518518518519,8.68480300187617,7.59683098591549,5.20892857142857,6.55690072639225,4.71813725490196,10.5326732673267,5.58238636363636,6.7728285077951,4.99133448873483,4.92066115702479,5.46954314720812,6.64876033057851,4.64333333333333,6.94103773584906,5.83661119515885,1.25507246376812,4.35823170731707,5.59943582510578,3.87025316455696,5.12820512820513,9.12230215827338,4.4263862332696,8.87857142857143,5.08571428571429,5.52156334231806,7.38687392055268,0.134952766531714,2.16666666666667,7.1342062193126,3.13576158940397,4.18789144050104,4.04119193689746,6.68,5.11254019292605,5.51532033426184,2.88752196836555,3.63636363636364,6.09014084507042,6.17587939698493,11.3463035019455,8.99860335195531,5.6,7.31612903225806,10.4351145038168,6.57305936073059,6.87959183673469,7.33082706766917,5.78097345132743,7.38961038961039,3.4299674267101,4.96419437340153,3.62758620689655,2.69801084990958,3.27642276422764,6.36914600550964,6.98688524590164,5.06089743589744,4.68586387434555,7.08481262327416,1,5.79683377308707,4.87064676616915,5.74803149606299,4.91495601173021,5.14823529411765,4.54559505409583,6.4525993883792,5.2697247706422,7.00472813238771,5.26170212765957,5.86346153846154,3.58552631578947,5.88979591836735,7.86949152542373,6.26571428571429,3.38195777351248,7.45353159851301,5.81914893617021,2.90889370932755,8.61176470588235,4.64943820224719,7.11677282377919,5.66726943942134,5.99540229885057,4.5993265993266,7.29573934837093,5.63404255319149,4.80582524271845,6.11444921316166,3.33333333333333,5.58605341246291,5.66230936819172,5.94072657743786,5.46444444444444,3.4419795221843,8.36027713625866,8.90277777777778,4.85288270377734,4.84285714285714,5.94893617021277,7.19875776397516,6.79795396419437,5.80112044817927,10.9859484777518,6,7.49859943977591,3.72139303482587,6.04379562043796,5.02040816326531,3.60734463276836,5.72244897959184,5.38654353562005,5.60189573459716,7.78571428571429,4.63812154696133,6.90161725067385,4.18529411764706,4.67582417582418,9.27142857142857,7.97005988023952,6.59510357815443,4.24064171122995,4.67729083665339,5.31450094161959,6.41680960548885,9.49602122015915,7.28239202657807,4.21652421652422,6.11830357142857,6.44280442804428,7.21676300578035,8.4187643020595,5.17777777777778,6.42073170731707,6.41681901279707,6.05112781954887,7.2972972972973,5.75165125495376,4.44599303135888,7.14111922141119,7.40986717267552,8.72361809045226,5.71638141809291,7.33727810650888,5.00995024875622,6.27695167286245,5.31353135313531,6.62247838616715,6.79964539007092,8.43718592964824,5.27513227513228,7.15497076023392,6.27733333333333,6.05089820359281,5.10725229826353,5.95290858725762,6.52341597796143,8.47721822541966,7.90536277602524,9.29945054945055,2.41257367387033,6.136,7.23453608247423,6.88888888888889,3.39114391143911,8.33842239185751,2.32057416267943,5.48314606741573,7.43414634146342,8.79166666666667,2.07834101382488,3.96332046332046,4.96782178217822,5.45849802371542,6.28165938864629,4.42942345924453,6.34868421052632,5.14071856287425,5.12594458438287,5.54144620811288,3.87017543859649,4.85893854748603,4.49719887955182,5.23076923076923,8.44933078393881,5.73190789473684,5.37100737100737,5.33394495412844,4.95652173913043,5.00726392251816,5.63326226012793,6.63938973647712,5.40331491712707,5.4792899408284,6.50219298245614,5.69371727748691,4.76727272727273,10.1424242424242,6.31306306306306,6.13690476190476,3.14765100671141,10.6554404145078,4.25117370892019,7.01806239737274,6.18811881188119,8.57692307692308,6.81565656565657,6.1520190023753,6.63464566929134,4.92038834951456,5.62518301610542,4.91312384473198,5.77994428969359,6.41810344827586,6.69248291571754,5.41538461538462,4.80508474576271,11.3714902807775,7.23126338329764,8.93766233766234,6.72065217391304,6.27015250544662,7.45322245322245,3.05350553505535,12.14453125,10.953488372093,5.53333333333333,4.84523809523809,7.26,3.35555555555556,3.8375350140056,6.68706293706294,7.27731092436975,6.03440860215054,5.30243161094225,6.66734279918864,8.56037151702786,7.3477537437604,5.21581196581197,2.04484304932735,5.70998116760829,3.53225806451613,8.71712158808933,4.52292020373514,6.37173913043478,3.63963963963964,6.43220338983051,6.88607594936709,9.16367076631977,5.45553822152886,4.09151414309484,4.15679442508711,6.68432671081678,6.14495798319328,3.07142857142857,5.51491053677932,7.77318295739348,4.48797250859107,5.82259663032706,2.91489361702128,9.51963048498845,6.10511363636364,5.85301837270341,5.88728323699422,6.42688679245283,5.224,4.79100529100529,4.46134020618557,10.1931216931217,2.6958904109589,5.13807531380753,4.68055555555556,2.70817120622568,5.79240506329114,5.58443708609272,3.68758526603001,7.47184986595174,4.94409937888199,7.14125200642055,5.95088408644401,6.30163447251114,4.45614035087719,4.34959349593496,2.45098039215686,6.58375634517767,5.67616191904048,3.36206896551724,2.76628352490421,5.125,7.34,5.45238095238095,4.62051282051282,5.26,7.13464696223317,9.3525,4.20382165605096,6.92724458204334,4.94468085106383,7.08387096774194,9.24500907441016,4.96495327102804,6.9890625,5.83692307692308,8.8125,1.19746835443038,4.9648033126294,6.47766323024055,6.34078212290503,5.74951076320939,7.83378746594005,5.70983213429257,1.76623376623377,2.3625,9.76363636363636,6.4327731092437,1,3.3961038961039,3.65192307692308,3.28623188405797,6.04694835680751,11.046130952381,7.47741935483871,5.46189024390244,1.92895204262877,5.05780346820809,4.3009900990099,4.84395604395604,8.48245614035088,4.57739938080495,8.38235294117647,5.292343387471,6.74373795761079,6.98954703832753,5.73200992555831,7.05760368663594,3.26258205689278,1.72674418604651,5.64931506849315,1.76388888888889,5.59880239520958,5.97704081632653,4.10416666666667,6.80970149253731,8.2756183745583,7.35166666666667,8.91666666666667,7.59081419624217,5.79205607476636,6.08315565031983,7.39210526315789,6.36430317848411,8.38992042440318,5.42746615087041,6.59144893111639,5.9,5.76470588235294,3.09423076923077,8.73365617433414,9.41208791208791,7.52450090744102,6.10055865921788,6.58088235294118,5.06206896551724,6.17905405405405,6.7134328358209,5.24362606232295,4.44201680672269,7.06073752711497,4.14656771799629,8.12459546925566,7.88513513513514,6.91495601173021,3.83662477558348,12.4585714285714,1,5.68836291913215,7.49560117302053,4.58333333333333,5.28688524590164,7.63823529411765,12.1771844660194,6.08507670850767,12.914,5.21531100478469,9.4089709762533,1.94827586206897,6.32451499118166,1,4.71288743882545,5.27152317880795,5.52008032128514,5.35740740740741,8.69955156950673,7.46439628482972,8.0920716112532,7.92058823529412,5.61804222648752,5.70299727520436,5.31223021582734,4.72911963882618,117.391304347826,4.3625,3.86029411764706,6.68535825545171,2.1437908496732,7.14285714285714,1.17857142857143,4.28700906344411,0.915422885572139,5.01111111111111,17.4,5.72796352583587,6.91909385113269,7.45844504021448,6.43298969072165,6.84158415841584,4.52086811352254,5.29263157894737,6.14646464646465,4.73429951690821,4.42682926829268,5.92352941176471,6.15637860082305,5.59250585480094,5.98863636363636,6.16444444444444,6.975,5.05504587155963,5.50608695652174,6.53703703703704,5.15631691648822,5.94818652849741,6.87664473684211,4.64864864864865,8.56028368794326,4.98456790123457,6.16308243727599,5.18840579710145,3.44107744107744,5.23333333333333,8.83898305084746,11.844298245614,5.20769230769231,4.13294797687861,2.94196428571429,7.01643835616438,5.62464985994398,4.72080291970803,6.26196473551637,5.2275204359673,7.2914691943128,6.11062906724512,6.41165413533835,6.09665427509294,7.05737704918033,4.59569377990431,6.62443438914027,6.55364806866953,6.57142857142857,4.21198156682028,5.67095115681234,6.55457227138643,4.73856209150327,5.56992084432718,1,1,6.06506849315068,5.83068783068783,6.85123966942149,4.86036036036036,5.77083333333333,5.48625429553265,4.36842105263158,6.14553990610329,4.25245098039216,6.1139646869984,5.89622641509434,6.55182926829268,10,7.04761904761905,9.01788908765653,9.5531914893617,1.70847457627119,4.53040540540541,4.76053639846743,7.54811715481172,8.39929947460595,6.55253623188406,1.26984126984127,4.73190348525469,4.93034055727554,6.75367647058824,6.77842565597668,5.32620320855615,2.90557939914163,4.4620886981402,6.03846153846154,4.01989389920424,12.8629032258065,5.55434782608696,8.05729166666667,5.31114808652246,3.8015873015873,6.67814371257485,5.67909238249595,8.04290429042904,6.18560606060606,5.54473161033797,4.82173913043478,6.46275071633238,5.09800664451827,6.95961995249406,5.69755244755245,5.39529914529915,1,1.06818181818182,2.25806451612903],"text":["Mean_Morphology_Major_Axis_Length:  30.588326<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  21.643376<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.707582<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.916786<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.110442<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.969564<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.887628<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.574978<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  44.814169<br />segmentation_metric:   1.4568966<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 102.319820<br />segmentation_metric:   9.0079787<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.901107<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.001246<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.906977<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.977386<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.536765<br />segmentation_metric:   1.0073529<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.788583<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 128.045299<br />segmentation_metric:   4.2817869<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  62.307653<br />segmentation_metric:   6.2910217<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.490101<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.153857<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.972604<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.574348<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.174627<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.819527<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.165410<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.319083<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.630328<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.374171<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  94.805598<br />segmentation_metric:   5.8403614<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  85.146587<br />segmentation_metric:   6.1716937<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  86.323846<br />segmentation_metric:   4.0149007<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  84.293901<br />segmentation_metric:   4.9822785<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  14.634773<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  23.621080<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 100.002066<br />segmentation_metric:   4.7355932<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 143.404970<br />segmentation_metric:   4.5383481<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  14.708782<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.386219<br />segmentation_metric:   4.3460899<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.585033<br />segmentation_metric:   4.1948998<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 119.221478<br />segmentation_metric:   4.6050584<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  50.942189<br />segmentation_metric:   3.3503788<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  82.038942<br />segmentation_metric:   3.1592129<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 129.731934<br />segmentation_metric:  12.9872123<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  41.311222<br />segmentation_metric:   1.4095238<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.474414<br />segmentation_metric:   4.8666667<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  92.200378<br />segmentation_metric:   6.1586207<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  57.498064<br />segmentation_metric:   5.7096774<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  75.895891<br />segmentation_metric:   6.0835165<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  33.734704<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.955449<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  58.135434<br />segmentation_metric:   4.1719198<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.885843<br />segmentation_metric:   1.9934211<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.578014<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  17.697890<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.647709<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.840836<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.336614<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  19.536252<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.683599<br />segmentation_metric:   3.3671875<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.521517<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.327123<br />segmentation_metric:   2.6111111<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 152.604953<br />segmentation_metric:   6.4934211<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.826307<br />segmentation_metric:   4.9242718<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  95.661206<br />segmentation_metric:   3.8086957<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  22.654649<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  36.596170<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  37.422177<br />segmentation_metric:   3.4342629<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.925947<br />segmentation_metric:   6.9253731<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 135.947480<br />segmentation_metric:   4.6647662<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  29.295462<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.291154<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  78.163577<br />segmentation_metric:   2.5952381<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  82.740121<br />segmentation_metric:   6.0521327<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.802741<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  54.953894<br />segmentation_metric:   3.1250000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  83.357727<br />segmentation_metric:   7.7625000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 151.077127<br />segmentation_metric:   6.0028736<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  35.866594<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  96.485016<br />segmentation_metric:   4.6561983<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  84.859545<br />segmentation_metric:   3.6354467<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.100922<br />segmentation_metric:   3.6538462<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  48.198030<br />segmentation_metric:   2.0567951<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  24.126280<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  37.744570<br />segmentation_metric:   2.3687316<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 112.097152<br />segmentation_metric:   6.9897436<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  90.072003<br />segmentation_metric:   3.3808140<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  66.469226<br />segmentation_metric:   3.2268212<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.806956<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  69.032152<br />segmentation_metric:   5.7801556<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.847621<br />segmentation_metric:   1.0606860<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.736577<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  25.896118<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  71.600166<br />segmentation_metric:   4.4125737<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  70.950040<br />segmentation_metric:   4.5817490<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.088653<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 117.727208<br />segmentation_metric:   6.4420804<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  46.470373<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  31.425151<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  20.434825<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  61.054576<br />segmentation_metric:   4.0841121<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 106.738445<br />segmentation_metric:   6.0506108<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  69.097240<br />segmentation_metric:   3.7439516<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  53.039420<br />segmentation_metric:   5.4542484<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  32.438375<br />segmentation_metric:   1.2845953<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  48.416001<br />segmentation_metric:   3.3373494<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  86.407747<br />segmentation_metric:   4.0610236<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.452570<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  34.256512<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  94.119105<br />segmentation_metric:   6.3560058<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  92.792455<br />segmentation_metric:   5.0056285<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  58.345869<br />segmentation_metric:   2.2665782<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  94.955127<br />segmentation_metric:   5.9631676<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 124.976135<br />segmentation_metric:   4.0906344<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.812966<br />segmentation_metric:   4.6357827<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 111.836445<br />segmentation_metric:   6.3358071<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 101.653736<br />segmentation_metric:   5.4116424<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  85.390281<br />segmentation_metric:   5.0930233<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 166.963783<br />segmentation_metric:   8.2917981<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  84.050455<br />segmentation_metric:   5.3966942<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.753570<br />segmentation_metric:   4.2109955<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  80.481993<br />segmentation_metric:   5.4369287<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  98.920918<br />segmentation_metric:   6.0976562<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  84.795959<br />segmentation_metric:   3.3972868<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  34.703614<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 101.318272<br />segmentation_metric:   4.5796915<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  26.481593<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  28.036374<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 104.856895<br />segmentation_metric:   9.4613734<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  79.748389<br />segmentation_metric:   6.9331897<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  65.641658<br />segmentation_metric:   7.7932011<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 108.032350<br />segmentation_metric:   6.8311688<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.681945<br />segmentation_metric:   7.8418972<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  74.611753<br />segmentation_metric:   7.2720764<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  83.814328<br />segmentation_metric:   5.8311688<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 100.180625<br />segmentation_metric:   7.1788079<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  88.353412<br />segmentation_metric:   4.0474308<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  81.781461<br />segmentation_metric:   8.1655844<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 130.704193<br />segmentation_metric:   8.7991573<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  52.762311<br />segmentation_metric:   5.6626506<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  59.873617<br />segmentation_metric:   5.9785933<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  73.621581<br />segmentation_metric:   6.3388430<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  69.465206<br />segmentation_metric:   6.5952381<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  92.077983<br />segmentation_metric:   6.9013333<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  41.598140<br />segmentation_metric:   1.5582329<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  60.728222<br />segmentation_metric:   5.8738462<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  91.370995<br />segmentation_metric:   8.1380145<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length: 111.851700<br />segmentation_metric:   9.0335097<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  58.356643<br />segmentation_metric:   3.6083551<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  30.638232<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 0","Mean_Morphology_Major_Axis_Length:  27.982896<br />segmentation_metric:   2.6569343<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 168.015317<br />segmentation_metric:   9.4382716<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  62.995803<br />segmentation_metric:   4.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.452980<br />segmentation_metric:   5.4654378<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.086120<br />segmentation_metric:   7.1957895<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.921205<br />segmentation_metric:   1.3983287<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   9.946350<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.002250<br />segmentation_metric:   4.6503497<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  19.856721<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.556526<br />segmentation_metric:   5.0776892<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  69.278903<br />segmentation_metric:   5.3899614<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  47.538694<br />segmentation_metric:   3.2046632<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.946337<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.574007<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  51.032326<br />segmentation_metric:   4.3471503<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  98.539513<br />segmentation_metric:   4.8796680<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.690364<br />segmentation_metric:   6.2458410<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  92.581007<br />segmentation_metric:   6.9254079<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  95.160343<br />segmentation_metric:   5.3530572<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.737418<br />segmentation_metric:   5.1392405<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  96.849551<br />segmentation_metric:   5.2160149<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 108.381632<br />segmentation_metric:   7.3189327<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.949334<br />segmentation_metric:   7.7525424<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  64.446349<br />segmentation_metric:   6.5250737<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 105.374021<br />segmentation_metric:   3.8694818<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.572760<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  36.716137<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  94.631980<br />segmentation_metric:   7.0530822<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  59.207506<br />segmentation_metric:   6.1950549<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  99.906576<br />segmentation_metric:   7.0262391<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  81.024119<br />segmentation_metric:   9.0782609<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.356746<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  80.789051<br />segmentation_metric:   8.9323308<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  34.279161<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  68.046277<br />segmentation_metric:   4.8825503<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  62.127543<br />segmentation_metric:   5.7567568<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  79.341901<br />segmentation_metric:   6.3650190<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.231410<br />segmentation_metric:   5.8042328<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  91.187080<br />segmentation_metric:  86.4242424<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  66.495936<br />segmentation_metric:   6.6147309<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  68.469787<br />segmentation_metric:   4.0880952<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.232914<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:   9.404353<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.280727<br />segmentation_metric:   7.4399185<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  83.432389<br />segmentation_metric:   6.7584034<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.877431<br />segmentation_metric:   6.1844444<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  80.038211<br />segmentation_metric:   5.2139738<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.005039<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.665674<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  88.153245<br />segmentation_metric:   7.0563636<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 112.314343<br />segmentation_metric:   9.7709091<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.952014<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  46.243902<br />segmentation_metric:   1.5823529<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  56.724711<br />segmentation_metric:   7.1909548<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.866187<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.777766<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  96.250682<br />segmentation_metric:   5.5890851<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.386730<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 103.662256<br />segmentation_metric:  11.1480769<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 120.771589<br />segmentation_metric:   4.5398833<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  75.832512<br />segmentation_metric:   3.6897275<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.340287<br />segmentation_metric:   1.2607595<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 129.048063<br />segmentation_metric:   8.3514851<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  85.080339<br />segmentation_metric:   4.5444126<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  57.698677<br />segmentation_metric:   4.3200000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  87.352190<br />segmentation_metric:   5.1147132<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  74.609388<br />segmentation_metric:   6.6950355<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  58.741101<br />segmentation_metric:   3.4097859<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  38.399508<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  36.240981<br />segmentation_metric:   1.0534682<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  23.232260<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.585586<br />segmentation_metric:   4.1539855<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  72.921128<br />segmentation_metric:   5.1563169<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 153.594370<br />segmentation_metric:   8.3338843<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  67.149965<br />segmentation_metric:   4.9230769<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 132.630752<br />segmentation_metric:   9.7389558<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  87.766785<br />segmentation_metric:   4.8969697<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.269932<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.646157<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.750029<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.493015<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.507578<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.765560<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  31.077138<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.561200<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.431179<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  29.557058<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.893915<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.365836<br />segmentation_metric:   3.1889597<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.468598<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  82.592224<br />segmentation_metric:   5.4673913<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.120758<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  50.956435<br />segmentation_metric:   2.1918736<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.103548<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  77.208190<br />segmentation_metric:   4.2509158<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  71.113022<br />segmentation_metric:   5.6459627<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  85.905974<br />segmentation_metric:   6.7032710<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  91.974950<br />segmentation_metric:   4.8725869<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 133.898340<br />segmentation_metric:   6.6101231<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.354851<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.496710<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.144674<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.466306<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  37.196526<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.942806<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  30.120521<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.944705<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  27.871443<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  28.558458<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.075611<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  32.744510<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  26.083181<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 112.877588<br />segmentation_metric:   7.1705989<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  33.903994<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  35.940732<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  96.133831<br />segmentation_metric:   6.3405088<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  25.455033<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  24.632478<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length: 122.912887<br />segmentation_metric:   3.8246493<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  61.248600<br />segmentation_metric:   5.6353276<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  76.012487<br />segmentation_metric:   7.0821918<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 1","Mean_Morphology_Major_Axis_Length:  89.690261<br />segmentation_metric:   9.7361111<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.017169<br />segmentation_metric:   7.5127737<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 123.519828<br />segmentation_metric:   9.8412322<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  81.972545<br />segmentation_metric:   6.2412060<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.784196<br />segmentation_metric:   6.0859107<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 120.945575<br />segmentation_metric:   7.1321696<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  79.739442<br />segmentation_metric:   5.9461358<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 110.802793<br />segmentation_metric:   5.2403361<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 119.095633<br />segmentation_metric:  10.9337748<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.183089<br />segmentation_metric:   7.0814978<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 123.604645<br />segmentation_metric:   6.7233273<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 168.697653<br />segmentation_metric:   8.5063521<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  76.630401<br />segmentation_metric:   8.3094059<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.132420<br />segmentation_metric:   5.3990499<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  23.624063<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.922235<br />segmentation_metric:   6.1102941<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  81.495612<br />segmentation_metric:   5.5027624<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.402438<br />segmentation_metric:   6.6147757<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.147527<br />segmentation_metric:   5.6330049<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 101.515107<br />segmentation_metric:   7.0352941<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  74.341078<br />segmentation_metric:   9.2189974<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.231800<br />segmentation_metric:   5.7954545<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  33.479204<br />segmentation_metric:   2.4915254<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  23.239425<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  36.156746<br />segmentation_metric:   2.8279221<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.563048<br />segmentation_metric:   6.7331461<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.672538<br />segmentation_metric:   5.4243986<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.893002<br />segmentation_metric:   8.5633270<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  71.134644<br />segmentation_metric:   5.3547237<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.374472<br />segmentation_metric:   7.9034608<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.724629<br />segmentation_metric:   5.1211573<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 104.209725<br />segmentation_metric:   7.2046875<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 105.907744<br />segmentation_metric:   5.3286119<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.832357<br />segmentation_metric:   4.4117647<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 241.846407<br />segmentation_metric:   7.6561086<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.992722<br />segmentation_metric:   5.7576471<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.505892<br />segmentation_metric:   8.1138952<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 165.802892<br />segmentation_metric:  12.2390852<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.773472<br />segmentation_metric:   7.3429319<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.265222<br />segmentation_metric:   6.9145729<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.841455<br />segmentation_metric:   4.8090349<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.780544<br />segmentation_metric:   6.5530973<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  95.519113<br />segmentation_metric:   5.6017897<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.462231<br />segmentation_metric:   5.7290837<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.064630<br />segmentation_metric:   7.1721854<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  72.267463<br />segmentation_metric:   4.9375000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 142.639458<br />segmentation_metric:   5.9090909<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.918789<br />segmentation_metric:   6.0728643<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.173407<br />segmentation_metric:   6.4709784<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  81.441944<br />segmentation_metric:   7.7493606<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.224205<br />segmentation_metric:   4.8051392<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.226976<br />segmentation_metric:   5.1369863<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.064978<br />segmentation_metric:   5.6652452<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  99.233142<br />segmentation_metric:   6.0895692<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  69.991934<br />segmentation_metric:   5.0349908<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.478385<br />segmentation_metric:   6.1267606<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 102.600233<br />segmentation_metric:   6.5315457<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.429074<br />segmentation_metric:   4.5471349<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 142.744876<br />segmentation_metric:   5.5956739<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.203703<br />segmentation_metric:   3.6380090<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  43.081842<br />segmentation_metric:   3.1735016<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.929533<br />segmentation_metric:   6.7472868<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  65.196109<br />segmentation_metric:   5.5917031<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.988979<br />segmentation_metric:   7.4034707<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  67.386309<br />segmentation_metric:   5.5334821<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  74.287311<br />segmentation_metric:   5.5458015<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.672812<br />segmentation_metric:   6.0840198<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 105.198002<br />segmentation_metric:   6.9572491<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.634365<br />segmentation_metric:   5.3926789<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.082556<br />segmentation_metric:   7.4810690<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  89.350163<br />segmentation_metric:   5.6540284<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 126.078208<br />segmentation_metric:   5.1084044<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.351506<br />segmentation_metric:  16.9929577<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  94.167760<br />segmentation_metric:   4.3292683<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 104.521379<br />segmentation_metric:   6.6819048<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  27.739471<br />segmentation_metric:   2.2146893<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 152.817186<br />segmentation_metric:   9.6101399<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.821633<br />segmentation_metric:   5.0828025<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  35.112418<br />segmentation_metric:   1.1734475<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  73.774951<br />segmentation_metric:   2.3050847<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  38.315528<br />segmentation_metric:   1.7601010<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.241647<br />segmentation_metric:   4.9048843<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 116.144540<br />segmentation_metric:  11.0220751<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 105.085350<br />segmentation_metric:   7.6295547<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 164.640221<br />segmentation_metric:   6.2996988<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.466857<br />segmentation_metric:   7.4507576<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.618908<br />segmentation_metric:   6.5466102<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  62.822444<br />segmentation_metric:   5.7631579<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.103797<br />segmentation_metric:   5.4830769<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  83.927732<br />segmentation_metric:   5.6723485<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 112.728716<br />segmentation_metric:   6.8624339<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.480952<br />segmentation_metric:   6.3560606<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.440155<br />segmentation_metric:   5.8664441<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  85.831260<br />segmentation_metric:   8.8474576<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.797709<br />segmentation_metric:   8.8990385<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 111.670437<br />segmentation_metric:   7.0759013<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  30.304676<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  93.785267<br />segmentation_metric:   6.2478448<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  80.563026<br />segmentation_metric:   7.1085271<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.220923<br />segmentation_metric:   8.1451876<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  99.113199<br />segmentation_metric:   6.5097493<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 105.253532<br />segmentation_metric:   6.5274262<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.109375<br />segmentation_metric:   7.4476190<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.852890<br />segmentation_metric:   7.8722467<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  36.809170<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  57.563533<br />segmentation_metric:   4.5116279<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 107.648812<br />segmentation_metric:   9.9581498<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.373388<br />segmentation_metric:   3.8304094<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.899841<br />segmentation_metric:   4.4365079<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  92.763767<br />segmentation_metric:   4.8199566<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  56.737854<br />segmentation_metric:   4.9742857<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 168.386685<br />segmentation_metric:   7.9535452<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  78.060021<br />segmentation_metric:   4.5922330<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  88.978876<br />segmentation_metric:   5.2834783<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  33.547872<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 103.563328<br />segmentation_metric:   5.6505017<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.335883<br />segmentation_metric:   5.4220000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 121.923773<br />segmentation_metric:   4.6960352<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  32.453618<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.802765<br />segmentation_metric:   5.8887172<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.145914<br />segmentation_metric:   5.3512195<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  97.884060<br />segmentation_metric:   4.9160959<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.360728<br />segmentation_metric:   4.8149171<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  77.722350<br />segmentation_metric:   4.5267176<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  27.997850<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 132.359472<br />segmentation_metric:   7.4385027<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.593637<br />segmentation_metric:   3.8090788<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.085051<br />segmentation_metric:   5.9772080<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 131.173938<br />segmentation_metric:   5.1355556<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 105.415181<br />segmentation_metric:   5.1445578<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  96.023571<br />segmentation_metric:   5.0173228<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  61.341317<br />segmentation_metric:   4.0904437<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  75.650080<br />segmentation_metric:   3.8741259<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.551475<br />segmentation_metric:   7.5045045<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  63.657219<br />segmentation_metric:   3.3559871<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  84.782278<br />segmentation_metric:   3.8347701<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  70.495974<br />segmentation_metric:   7.7876923<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 128.898862<br />segmentation_metric:   5.4249548<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  64.844200<br />segmentation_metric:   6.5828221<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  86.646628<br />segmentation_metric:   7.4379947<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  98.448155<br />segmentation_metric:   4.9854015<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  91.271707<br />segmentation_metric:   3.4458599<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  87.167700<br />segmentation_metric:   7.2097187<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  49.229216<br />segmentation_metric:   2.7208791<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  82.915077<br />segmentation_metric:   3.7029289<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  68.775514<br />segmentation_metric:   6.6789838<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.961561<br />segmentation_metric:   6.1620553<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  55.815653<br />segmentation_metric:   3.8768657<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 100.541014<br />segmentation_metric:   6.4530831<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length:  32.236744<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 2","Mean_Morphology_Major_Axis_Length: 106.032881<br />segmentation_metric:   6.2993631<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  35.979954<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 164.674521<br />segmentation_metric:   5.7655502<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.921956<br />segmentation_metric:   4.1071429<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.797522<br />segmentation_metric:   6.7240664<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.004074<br />segmentation_metric:   6.6134615<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.497863<br />segmentation_metric:   5.5469314<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  93.160829<br />segmentation_metric:   6.1079545<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 132.453308<br />segmentation_metric:   7.7463054<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.713305<br />segmentation_metric:   5.0090323<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  94.237616<br />segmentation_metric:   7.4604743<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.094807<br />segmentation_metric:   3.9404990<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.443231<br />segmentation_metric:   5.7480916<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.345516<br />segmentation_metric:   5.1028446<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 109.641144<br />segmentation_metric:   6.7799296<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  90.574627<br />segmentation_metric:   6.6590909<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.592962<br />segmentation_metric:   5.6648045<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 113.324782<br />segmentation_metric:   9.8309091<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 104.296069<br />segmentation_metric:   6.2377049<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.686351<br />segmentation_metric:   7.2254902<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 111.964131<br />segmentation_metric:   6.4947839<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 109.367137<br />segmentation_metric:   5.7496136<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  52.263401<br />segmentation_metric:   3.6052632<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 125.812548<br />segmentation_metric:   8.4288793<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 111.314533<br />segmentation_metric:   5.1759891<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  45.871978<br />segmentation_metric:   2.9502924<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 172.125261<br />segmentation_metric:   8.3048128<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 112.018144<br />segmentation_metric:   9.2629310<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.590785<br />segmentation_metric:   4.6628788<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.820393<br />segmentation_metric:   4.9032258<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  93.539689<br />segmentation_metric:   5.0834915<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.500490<br />segmentation_metric:   8.3167082<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.181412<br />segmentation_metric:   6.0189983<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  73.733983<br />segmentation_metric:   5.0447368<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.863815<br />segmentation_metric:   3.6526316<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.508493<br />segmentation_metric:   5.8604651<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  57.431230<br />segmentation_metric:   4.0757576<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 104.204507<br />segmentation_metric:   9.0375817<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.996149<br />segmentation_metric:   6.6790451<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 114.861934<br />segmentation_metric:   7.1098485<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.018761<br />segmentation_metric:   5.3622291<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.915263<br />segmentation_metric:   5.4532578<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.232281<br />segmentation_metric:   7.0873147<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.267933<br />segmentation_metric:   5.0848586<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.856435<br />segmentation_metric:   5.0490463<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 155.811055<br />segmentation_metric:  10.8214748<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  36.108105<br />segmentation_metric:   1.0681818<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.215236<br />segmentation_metric:   5.3688312<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  47.511415<br />segmentation_metric:   6.8461538<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.546899<br />segmentation_metric:   5.1574074<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  24.510129<br />segmentation_metric:   2.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.857738<br />segmentation_metric:   5.0591549<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  90.747922<br />segmentation_metric:   4.7961373<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 106.990641<br />segmentation_metric:   8.2801252<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  68.209916<br />segmentation_metric:   5.6925926<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.336084<br />segmentation_metric:   6.1906077<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.538501<br />segmentation_metric:   6.4120879<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  65.284470<br />segmentation_metric:   5.4513514<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.056613<br />segmentation_metric:   6.2278820<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  89.319439<br />segmentation_metric:   7.7367206<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  81.152583<br />segmentation_metric:   5.7004049<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  49.152231<br />segmentation_metric:   3.5822222<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.180443<br />segmentation_metric:   7.2868369<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  98.869313<br />segmentation_metric:   7.0066556<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  63.492467<br />segmentation_metric:   5.4914894<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.581777<br />segmentation_metric:   8.2666667<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  76.689856<br />segmentation_metric:   6.2179775<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.223872<br />segmentation_metric:   6.5000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 108.205993<br />segmentation_metric:  10.9387755<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 113.467444<br />segmentation_metric:   6.2175732<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  90.823598<br />segmentation_metric:  11.3111702<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   9.147286<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.852354<br />segmentation_metric:   8.2623762<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  65.401114<br />segmentation_metric:   6.6807388<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  79.977300<br />segmentation_metric:   5.5442043<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 117.035279<br />segmentation_metric:   6.2835052<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.039949<br />segmentation_metric:   5.9058172<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 105.147423<br />segmentation_metric:   6.4618056<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.864677<br />segmentation_metric:   6.6851852<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.159493<br />segmentation_metric:   8.6848030<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 109.246635<br />segmentation_metric:   7.5968310<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  75.225335<br />segmentation_metric:   5.2089286<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 108.450795<br />segmentation_metric:   6.5569007<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.177032<br />segmentation_metric:   4.7181373<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.312578<br />segmentation_metric:  10.5326733<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  53.025302<br />segmentation_metric:   5.5823864<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 116.370280<br />segmentation_metric:   6.7728285<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  93.037452<br />segmentation_metric:   4.9913345<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.747938<br />segmentation_metric:   4.9206612<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  92.142670<br />segmentation_metric:   5.4695431<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 107.754916<br />segmentation_metric:   6.6487603<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  57.051071<br />segmentation_metric:   4.6433333<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  90.452265<br />segmentation_metric:   6.9410377<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 134.861579<br />segmentation_metric:   5.8366112<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  27.292427<br />segmentation_metric:   1.2550725<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  99.489098<br />segmentation_metric:   4.3582317<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  97.719166<br />segmentation_metric:   5.5994358<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  46.862213<br />segmentation_metric:   3.8702532<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.052699<br />segmentation_metric:   5.1282051<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 119.895164<br />segmentation_metric:   9.1223022<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.602210<br />segmentation_metric:   4.4263862<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 170.193993<br />segmentation_metric:   8.8785714<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.045385<br />segmentation_metric:   5.0857143<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.830096<br />segmentation_metric:   5.5215633<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 119.717545<br />segmentation_metric:   7.3868739<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   9.345983<br />segmentation_metric:   0.1349528<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  41.878714<br />segmentation_metric:   2.1666667<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 107.922809<br />segmentation_metric:   7.1342062<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  37.082303<br />segmentation_metric:   3.1357616<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  62.641793<br />segmentation_metric:   4.1878914<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 114.866851<br />segmentation_metric:   4.0411919<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.741089<br />segmentation_metric:   6.6800000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.258469<br />segmentation_metric:   5.1125402<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.555295<br />segmentation_metric:   5.5153203<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  60.522137<br />segmentation_metric:   2.8875220<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  73.973158<br />segmentation_metric:   3.6363636<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  95.725976<br />segmentation_metric:   6.0901408<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.733828<br />segmentation_metric:   6.1758794<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  75.238636<br />segmentation_metric:  11.3463035<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 141.993301<br />segmentation_metric:   8.9986034<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  66.770472<br />segmentation_metric:   5.6000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  88.696290<br />segmentation_metric:   7.3161290<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 138.017318<br />segmentation_metric:  10.4351145<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  80.589728<br />segmentation_metric:   6.5730594<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.241101<br />segmentation_metric:   6.8795918<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.148107<br />segmentation_metric:   7.3308271<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  74.386873<br />segmentation_metric:   5.7809735<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 123.523098<br />segmentation_metric:   7.3896104<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  53.961820<br />segmentation_metric:   3.4299674<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.379323<br />segmentation_metric:   4.9641944<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  35.690264<br />segmentation_metric:   3.6275862<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.874942<br />segmentation_metric:   2.6980108<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  25.207880<br />segmentation_metric:   3.2764228<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.944527<br />segmentation_metric:   6.3691460<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  65.175007<br />segmentation_metric:   6.9868852<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  52.859581<br />segmentation_metric:   5.0608974<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  71.927387<br />segmentation_metric:   4.6858639<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  77.807900<br />segmentation_metric:   7.0848126<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:   6.353278<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.239804<br />segmentation_metric:   5.7968338<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  70.603975<br />segmentation_metric:   4.8706468<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  83.807893<br />segmentation_metric:   5.7480315<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  82.161458<br />segmentation_metric:   4.9149560<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 111.222661<br />segmentation_metric:   5.1482353<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  91.180372<br />segmentation_metric:   4.5455951<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  59.471799<br />segmentation_metric:   6.4525994<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 102.950314<br />segmentation_metric:   5.2697248<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  86.336996<br />segmentation_metric:   7.0047281<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  75.076154<br />segmentation_metric:   5.2617021<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.893789<br />segmentation_metric:   5.8634615<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  67.524191<br />segmentation_metric:   3.5855263<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.051273<br />segmentation_metric:   5.8897959<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 112.843434<br />segmentation_metric:   7.8694915<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  96.828838<br />segmentation_metric:   6.2657143<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  87.142596<br />segmentation_metric:   3.3819578<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length: 100.570149<br />segmentation_metric:   7.4535316<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  70.753211<br />segmentation_metric:   5.8191489<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  56.845147<br />segmentation_metric:   2.9088937<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 3","Mean_Morphology_Major_Axis_Length:  78.301792<br />segmentation_metric:   8.6117647<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.758333<br />segmentation_metric:   4.6494382<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.983185<br />segmentation_metric:   7.1167728<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 128.346060<br />segmentation_metric:   5.6672694<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  86.591201<br />segmentation_metric:   5.9954023<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 110.336455<br />segmentation_metric:   4.5993266<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  71.480266<br />segmentation_metric:   7.2957393<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.644724<br />segmentation_metric:   5.6340426<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.637606<br />segmentation_metric:   4.8058252<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 112.948904<br />segmentation_metric:   6.1144492<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.635382<br />segmentation_metric:   3.3333333<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  87.280830<br />segmentation_metric:   5.5860534<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  90.971911<br />segmentation_metric:   5.6623094<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 177.024643<br />segmentation_metric:   5.9407266<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 106.921014<br />segmentation_metric:   5.4644444<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.093068<br />segmentation_metric:   3.4419795<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 149.709660<br />segmentation_metric:   8.3602771<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 123.929060<br />segmentation_metric:   8.9027778<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.429086<br />segmentation_metric:   4.8528827<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  89.747572<br />segmentation_metric:   4.8428571<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  92.951007<br />segmentation_metric:   5.9489362<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.003706<br />segmentation_metric:   7.1987578<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  74.813521<br />segmentation_metric:   6.7979540<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  73.384499<br />segmentation_metric:   5.8011204<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 102.201773<br />segmentation_metric:  10.9859485<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  85.528290<br />segmentation_metric:   6.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.116427<br />segmentation_metric:   7.4985994<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  48.387289<br />segmentation_metric:   3.7213930<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.532291<br />segmentation_metric:   6.0437956<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.383700<br />segmentation_metric:   5.0204082<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  51.154286<br />segmentation_metric:   3.6073446<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.258370<br />segmentation_metric:   5.7224490<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 127.243733<br />segmentation_metric:   5.3865435<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.601421<br />segmentation_metric:   5.6018957<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 139.228680<br />segmentation_metric:   7.7857143<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  59.759593<br />segmentation_metric:   4.6381215<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.211894<br />segmentation_metric:   6.9016173<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  55.161175<br />segmentation_metric:   4.1852941<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.430422<br />segmentation_metric:   4.6758242<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 107.128742<br />segmentation_metric:   9.2714286<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  91.961377<br />segmentation_metric:   7.9700599<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 114.685855<br />segmentation_metric:   6.5951036<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  48.656574<br />segmentation_metric:   4.2406417<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 120.689951<br />segmentation_metric:   4.6772908<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 133.956390<br />segmentation_metric:   5.3145009<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 137.264449<br />segmentation_metric:   6.4168096<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  74.642808<br />segmentation_metric:   9.4960212<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 151.060097<br />segmentation_metric:   7.2823920<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  52.298476<br />segmentation_metric:   4.2165242<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.568266<br />segmentation_metric:   6.1183036<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.297928<br />segmentation_metric:   6.4428044<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 244.997092<br />segmentation_metric:   7.2167630<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 106.436077<br />segmentation_metric:   8.4187643<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.980523<br />segmentation_metric:   5.1777778<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 116.203870<br />segmentation_metric:   6.4207317<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 130.313188<br />segmentation_metric:   6.4168190<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 108.713180<br />segmentation_metric:   6.0511278<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  87.352670<br />segmentation_metric:   7.2972973<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 109.909689<br />segmentation_metric:   5.7516513<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.833489<br />segmentation_metric:   4.4459930<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 102.024165<br />segmentation_metric:   7.1411192<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 136.061306<br />segmentation_metric:   7.4098672<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 109.784672<br />segmentation_metric:   8.7236181<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  80.327953<br />segmentation_metric:   5.7163814<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.795161<br />segmentation_metric:   7.3372781<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  63.969831<br />segmentation_metric:   5.0099502<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  82.026154<br />segmentation_metric:   6.2769517<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.800650<br />segmentation_metric:   5.3135314<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 108.448152<br />segmentation_metric:   6.6224784<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.103956<br />segmentation_metric:   6.7996454<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 246.657970<br />segmentation_metric:   8.4371859<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  68.535259<br />segmentation_metric:   5.2751323<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 116.410028<br />segmentation_metric:   7.1549708<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.862494<br />segmentation_metric:   6.2773333<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  61.852985<br />segmentation_metric:   6.0508982<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 100.921154<br />segmentation_metric:   5.1072523<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  68.139218<br />segmentation_metric:   5.9529086<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  63.872313<br />segmentation_metric:   6.5234160<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  95.381896<br />segmentation_metric:   8.4772182<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.930455<br />segmentation_metric:   7.9053628<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.652226<br />segmentation_metric:   9.2994505<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  57.231870<br />segmentation_metric:   2.4125737<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  78.637393<br />segmentation_metric:   6.1360000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.916809<br />segmentation_metric:   7.2345361<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.705827<br />segmentation_metric:   6.8888889<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  41.652629<br />segmentation_metric:   3.3911439<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 123.364050<br />segmentation_metric:   8.3384224<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  31.307426<br />segmentation_metric:   2.3205742<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  75.071527<br />segmentation_metric:   5.4831461<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.626041<br />segmentation_metric:   7.4341463<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  78.355936<br />segmentation_metric:   8.7916667<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  30.129776<br />segmentation_metric:   2.0783410<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  60.819582<br />segmentation_metric:   3.9633205<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  71.516064<br />segmentation_metric:   4.9678218<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 108.745071<br />segmentation_metric:   5.4584980<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 141.550920<br />segmentation_metric:   6.2816594<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  87.729898<br />segmentation_metric:   4.4294235<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.298696<br />segmentation_metric:   6.3486842<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 102.246499<br />segmentation_metric:   5.1407186<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  88.824738<br />segmentation_metric:   5.1259446<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 102.857658<br />segmentation_metric:   5.5414462<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  99.716283<br />segmentation_metric:   3.8701754<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.711278<br />segmentation_metric:   4.8589385<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.051498<br />segmentation_metric:   4.4971989<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 100.490774<br />segmentation_metric:   5.2307692<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  94.648943<br />segmentation_metric:   8.4493308<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 103.440594<br />segmentation_metric:   5.7319079<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  69.614595<br />segmentation_metric:   5.3710074<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  67.332401<br />segmentation_metric:   5.3339450<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.868987<br />segmentation_metric:   4.9565217<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  76.453636<br />segmentation_metric:   5.0072639<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.850290<br />segmentation_metric:   5.6332623<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 147.601862<br />segmentation_metric:   6.6393897<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.898409<br />segmentation_metric:   5.4033149<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  59.499128<br />segmentation_metric:   5.4792899<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 143.388948<br />segmentation_metric:   6.5021930<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  72.315163<br />segmentation_metric:   5.6937173<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  65.829585<br />segmentation_metric:   4.7672727<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.100144<br />segmentation_metric:  10.1424242<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.450510<br />segmentation_metric:   6.3130631<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  55.452122<br />segmentation_metric:   6.1369048<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  74.563703<br />segmentation_metric:   3.1476510<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 126.040551<br />segmentation_metric:  10.6554404<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  66.396849<br />segmentation_metric:   4.2511737<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 122.452040<br />segmentation_metric:   7.0180624<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  70.040464<br />segmentation_metric:   6.1881188<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  73.198512<br />segmentation_metric:   8.5769231<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.676683<br />segmentation_metric:   6.8156566<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  71.434923<br />segmentation_metric:   6.1520190<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  81.650717<br />segmentation_metric:   6.6346457<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  77.220749<br />segmentation_metric:   4.9203883<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  83.486783<br />segmentation_metric:   5.6251830<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  93.942684<br />segmentation_metric:   4.9131238<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  57.606863<br />segmentation_metric:   5.7799443<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length:  96.637411<br />segmentation_metric:   6.4181034<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 128.713304<br />segmentation_metric:   6.6924829<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 120.141567<br />segmentation_metric:   5.4153846<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 101.867147<br />segmentation_metric:   4.8050847<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 4","Mean_Morphology_Major_Axis_Length: 171.442979<br />segmentation_metric:  11.3714903<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 108.328109<br />segmentation_metric:   7.2312634<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.166577<br />segmentation_metric:   8.9376623<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 139.137327<br />segmentation_metric:   6.7206522<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.448456<br />segmentation_metric:   6.2701525<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.387825<br />segmentation_metric:   7.4532225<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  92.325704<br />segmentation_metric:   3.0535055<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 136.514046<br />segmentation_metric:  12.1445312<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 118.919374<br />segmentation_metric:  10.9534884<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 124.006805<br />segmentation_metric:   5.5333333<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.602822<br />segmentation_metric:   4.8452381<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.021322<br />segmentation_metric:   7.2600000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  53.602520<br />segmentation_metric:   3.3555556<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  68.594523<br />segmentation_metric:   3.8375350<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 171.108432<br />segmentation_metric:   6.6870629<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  72.589920<br />segmentation_metric:   7.2773109<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.428848<br />segmentation_metric:   6.0344086<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 173.502456<br />segmentation_metric:   5.3024316<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.291648<br />segmentation_metric:   6.6673428<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.246416<br />segmentation_metric:   8.5603715<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 114.076130<br />segmentation_metric:   7.3477537<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  71.484524<br />segmentation_metric:   5.2158120<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  26.743697<br />segmentation_metric:   2.0448430<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  74.879231<br />segmentation_metric:   5.7099812<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  92.248949<br />segmentation_metric:   3.5322581<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 110.088103<br />segmentation_metric:   8.7171216<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  86.393464<br />segmentation_metric:   4.5229202<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.711899<br />segmentation_metric:   6.3717391<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.795084<br />segmentation_metric:   3.6396396<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 131.959020<br />segmentation_metric:   6.4322034<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.461968<br />segmentation_metric:   6.8860759<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 161.980344<br />segmentation_metric:   9.1636708<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 127.912643<br />segmentation_metric:   5.4555382<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.811524<br />segmentation_metric:   4.0915141<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  44.129145<br />segmentation_metric:   4.1567944<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  88.860524<br />segmentation_metric:   6.6843267<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.637402<br />segmentation_metric:   6.1449580<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  45.968509<br />segmentation_metric:   3.0714286<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.653172<br />segmentation_metric:   5.5149105<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 124.325588<br />segmentation_metric:   7.7731830<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  69.027367<br />segmentation_metric:   4.4879725<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 137.518695<br />segmentation_metric:   5.8225966<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  41.945602<br />segmentation_metric:   2.9148936<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 118.710405<br />segmentation_metric:   9.5196305<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  70.030248<br />segmentation_metric:   6.1051136<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 118.068847<br />segmentation_metric:   5.8530184<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.556614<br />segmentation_metric:   5.8872832<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 116.558922<br />segmentation_metric:   6.4268868<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.142124<br />segmentation_metric:   5.2240000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  51.566033<br />segmentation_metric:   4.7910053<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  62.253965<br />segmentation_metric:   4.4613402<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  98.428209<br />segmentation_metric:  10.1931217<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  49.385392<br />segmentation_metric:   2.6958904<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.578054<br />segmentation_metric:   5.1380753<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  47.918959<br />segmentation_metric:   4.6805556<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  44.430616<br />segmentation_metric:   2.7081712<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.260547<br />segmentation_metric:   5.7924051<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 111.021418<br />segmentation_metric:   5.5844371<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 100.057662<br />segmentation_metric:   3.6875853<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.421729<br />segmentation_metric:   7.4718499<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  75.637820<br />segmentation_metric:   4.9440994<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 132.991968<br />segmentation_metric:   7.1412520<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.255592<br />segmentation_metric:   5.9508841<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.742502<br />segmentation_metric:   6.3016345<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  62.850066<br />segmentation_metric:   4.4561404<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  76.916143<br />segmentation_metric:   4.3495935<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  35.774611<br />segmentation_metric:   2.4509804<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.913929<br />segmentation_metric:   6.5837563<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  83.974840<br />segmentation_metric:   5.6761619<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  43.570110<br />segmentation_metric:   3.3620690<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  38.143836<br />segmentation_metric:   2.7662835<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.694486<br />segmentation_metric:   5.1250000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 123.511574<br />segmentation_metric:   7.3400000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.141721<br />segmentation_metric:   5.4523810<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 103.077461<br />segmentation_metric:   4.6205128<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 102.251419<br />segmentation_metric:   5.2600000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 111.493164<br />segmentation_metric:   7.1346470<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.521825<br />segmentation_metric:   9.3525000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  53.976544<br />segmentation_metric:   4.2038217<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.855901<br />segmentation_metric:   6.9272446<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 116.558076<br />segmentation_metric:   4.9446809<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.667154<br />segmentation_metric:   7.0838710<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 158.792670<br />segmentation_metric:   9.2450091<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  63.150014<br />segmentation_metric:   4.9649533<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 107.897673<br />segmentation_metric:   6.9890625<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  53.551715<br />segmentation_metric:   5.8369231<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 116.452968<br />segmentation_metric:   8.8125000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  31.387968<br />segmentation_metric:   1.1974684<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 113.614942<br />segmentation_metric:   4.9648033<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  86.719539<br />segmentation_metric:   6.4776632<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  63.973925<br />segmentation_metric:   6.3407821<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  95.354109<br />segmentation_metric:   5.7495108<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.207764<br />segmentation_metric:   7.8337875<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  64.852515<br />segmentation_metric:   5.7098321<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  22.995306<br />segmentation_metric:   1.7662338<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  50.893986<br />segmentation_metric:   2.3625000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.241151<br />segmentation_metric:   9.7636364<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  96.383914<br />segmentation_metric:   6.4327731<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:   4.472136<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  38.943120<br />segmentation_metric:   3.3961039<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  57.429343<br />segmentation_metric:   3.6519231<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  48.821370<br />segmentation_metric:   3.2862319<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  79.471455<br />segmentation_metric:   6.0469484<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 117.447613<br />segmentation_metric:  11.0461310<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 104.647820<br />segmentation_metric:   7.4774194<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.037685<br />segmentation_metric:   5.4618902<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  44.897054<br />segmentation_metric:   1.9289520<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  59.762215<br />segmentation_metric:   5.0578035<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  86.864106<br />segmentation_metric:   4.3009901<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.465823<br />segmentation_metric:   4.8439560<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 180.142301<br />segmentation_metric:   8.4824561<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 111.450592<br />segmentation_metric:   4.5773994<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.834105<br />segmentation_metric:   8.3823529<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.207678<br />segmentation_metric:   5.2923434<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  89.722002<br />segmentation_metric:   6.7437380<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  77.551622<br />segmentation_metric:   6.9895470<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  90.383366<br />segmentation_metric:   5.7320099<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.298515<br />segmentation_metric:   7.0576037<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  54.146353<br />segmentation_metric:   3.2625821<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  23.080442<br />segmentation_metric:   1.7267442<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  76.031116<br />segmentation_metric:   5.6493151<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  21.265981<br />segmentation_metric:   1.7638889<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  78.638582<br />segmentation_metric:   5.5988024<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  69.720033<br />segmentation_metric:   5.9770408<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  48.198661<br />segmentation_metric:   4.1041667<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.564659<br />segmentation_metric:   6.8097015<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 109.968638<br />segmentation_metric:   8.2756184<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  92.874362<br />segmentation_metric:   7.3516667<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  81.840329<br />segmentation_metric:   8.9166667<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.243185<br />segmentation_metric:   7.5908142<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.618617<br />segmentation_metric:   5.7920561<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  67.364873<br />segmentation_metric:   6.0831557<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  91.373056<br />segmentation_metric:   7.3921053<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  80.683931<br />segmentation_metric:   6.3643032<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  87.639631<br />segmentation_metric:   8.3899204<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 105.791391<br />segmentation_metric:   5.4274662<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  85.339458<br />segmentation_metric:   6.5914489<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 111.111034<br />segmentation_metric:   5.9000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  62.964544<br />segmentation_metric:   5.7647059<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  55.746632<br />segmentation_metric:   3.0942308<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 108.961315<br />segmentation_metric:   8.7336562<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 128.797123<br />segmentation_metric:   9.4120879<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  82.584384<br />segmentation_metric:   7.5245009<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  76.217201<br />segmentation_metric:   6.1005587<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length: 154.377421<br />segmentation_metric:   6.5808824<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.528243<br />segmentation_metric:   5.0620690<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  84.055965<br />segmentation_metric:   6.1790541<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  60.799704<br />segmentation_metric:   6.7134328<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 5","Mean_Morphology_Major_Axis_Length:  79.095484<br />segmentation_metric:   5.2436261<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  71.197137<br />segmentation_metric:   4.4420168<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.483229<br />segmentation_metric:   7.0607375<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.313272<br />segmentation_metric:   4.1465677<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 124.700798<br />segmentation_metric:   8.1245955<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.250964<br />segmentation_metric:   7.8851351<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  60.910229<br />segmentation_metric:   6.9149560<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  71.705365<br />segmentation_metric:   3.8366248<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 114.920358<br />segmentation_metric:  12.4585714<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:   5.437579<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  68.927319<br />segmentation_metric:   5.6883629<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.190718<br />segmentation_metric:   7.4956012<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  92.469432<br />segmentation_metric:   4.5833333<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  51.175747<br />segmentation_metric:   5.2868852<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  79.696120<br />segmentation_metric:   7.6382353<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 140.476332<br />segmentation_metric:  12.1771845<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.670527<br />segmentation_metric:   6.0850767<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.932239<br />segmentation_metric:  12.9140000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 142.265348<br />segmentation_metric:   5.2153110<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  86.417594<br />segmentation_metric:   9.4089710<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  18.256201<br />segmentation_metric:   1.9482759<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 138.290109<br />segmentation_metric:   6.3245150<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:   6.324610<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 121.206390<br />segmentation_metric:   4.7128874<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  68.833990<br />segmentation_metric:   5.2715232<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.118813<br />segmentation_metric:   5.5200803<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.371048<br />segmentation_metric:   5.3574074<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  95.488487<br />segmentation_metric:   8.6995516<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  69.757680<br />segmentation_metric:   7.4643963<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 137.180277<br />segmentation_metric:   8.0920716<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.295956<br />segmentation_metric:   7.9205882<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.994555<br />segmentation_metric:   5.6180422<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.079524<br />segmentation_metric:   5.7029973<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.903057<br />segmentation_metric:   5.3122302<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  58.036477<br />segmentation_metric:   4.7291196<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.948453<br />segmentation_metric: 117.3913043<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  42.007676<br />segmentation_metric:   4.3625000<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  51.582408<br />segmentation_metric:   3.8602941<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  54.388230<br />segmentation_metric:   6.6853583<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  25.167547<br />segmentation_metric:   2.1437908<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  16.249092<br />segmentation_metric:   7.1428571<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:   9.395695<br />segmentation_metric:   1.1785714<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.012834<br />segmentation_metric:   4.2870091<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  12.406573<br />segmentation_metric:   0.9154229<br />well_name: G09 <br>well_pos_x: 0 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.433001<br />segmentation_metric:   5.0111111<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 113.417097<br />segmentation_metric:  17.4000000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 134.363653<br />segmentation_metric:   5.7279635<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.533475<br />segmentation_metric:   6.9190939<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  73.157457<br />segmentation_metric:   7.4584450<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  90.772813<br />segmentation_metric:   6.4329897<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 144.378709<br />segmentation_metric:   6.8415842<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 195.327650<br />segmentation_metric:   4.5208681<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.880003<br />segmentation_metric:   5.2926316<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 118.694761<br />segmentation_metric:   6.1464646<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  58.218516<br />segmentation_metric:   4.7342995<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.520600<br />segmentation_metric:   4.4268293<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.392921<br />segmentation_metric:   5.9235294<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.988717<br />segmentation_metric:   6.1563786<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.531465<br />segmentation_metric:   5.5925059<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 117.443473<br />segmentation_metric:   5.9886364<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  68.930337<br />segmentation_metric:   6.1644444<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.117019<br />segmentation_metric:   6.9750000<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 122.304928<br />segmentation_metric:   5.0550459<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.967158<br />segmentation_metric:   5.5060870<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.619177<br />segmentation_metric:   6.5370370<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.152217<br />segmentation_metric:   5.1563169<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  62.004160<br />segmentation_metric:   5.9481865<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  84.333895<br />segmentation_metric:   6.8766447<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  56.896740<br />segmentation_metric:   4.6486486<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.009604<br />segmentation_metric:   8.5602837<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  53.230712<br />segmentation_metric:   4.9845679<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 134.894619<br />segmentation_metric:   6.1630824<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.028914<br />segmentation_metric:   5.1884058<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  40.348587<br />segmentation_metric:   3.4410774<br />well_name: G09 <br>well_pos_x: 1 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 103.100903<br />segmentation_metric:   5.2333333<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  94.500617<br />segmentation_metric:   8.8389831<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 157.435203<br />segmentation_metric:  11.8442982<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  89.049724<br />segmentation_metric:   5.2076923<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  42.693166<br />segmentation_metric:   4.1329480<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  43.883568<br />segmentation_metric:   2.9419643<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.244072<br />segmentation_metric:   7.0164384<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  59.781352<br />segmentation_metric:   5.6246499<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  80.859164<br />segmentation_metric:   4.7208029<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.828890<br />segmentation_metric:   6.2619647<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  95.985768<br />segmentation_metric:   5.2275204<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.059027<br />segmentation_metric:   7.2914692<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 103.396420<br />segmentation_metric:   6.1106291<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 105.475690<br />segmentation_metric:   6.4116541<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  97.768303<br />segmentation_metric:   6.0966543<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  78.772768<br />segmentation_metric:   7.0573770<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  84.995395<br />segmentation_metric:   4.5956938<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 136.030410<br />segmentation_metric:   6.6244344<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 247.175591<br />segmentation_metric:   6.5536481<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  99.644206<br />segmentation_metric:   6.5714286<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.290124<br />segmentation_metric:   4.2119816<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.849342<br />segmentation_metric:   5.6709512<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  85.988925<br />segmentation_metric:   6.5545723<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.829635<br />segmentation_metric:   4.7385621<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  65.442878<br />segmentation_metric:   5.5699208<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  15.792389<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  16.990421<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 2 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.352977<br />segmentation_metric:   6.0650685<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.565576<br />segmentation_metric:   5.8306878<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  63.365458<br />segmentation_metric:   6.8512397<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  66.234337<br />segmentation_metric:   4.8603604<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.257939<br />segmentation_metric:   5.7708333<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.416598<br />segmentation_metric:   5.4862543<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  35.436120<br />segmentation_metric:   4.3684211<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  65.098763<br />segmentation_metric:   6.1455399<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.619572<br />segmentation_metric:   4.2524510<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 106.999432<br />segmentation_metric:   6.1139647<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  64.222338<br />segmentation_metric:   5.8962264<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 124.212862<br />segmentation_metric:   6.5518293<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 129.466099<br />segmentation_metric:  10.0000000<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  96.709357<br />segmentation_metric:   7.0476190<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 148.691444<br />segmentation_metric:   9.0178891<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.283077<br />segmentation_metric:   9.5531915<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  26.043506<br />segmentation_metric:   1.7084746<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  52.740683<br />segmentation_metric:   4.5304054<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.033515<br />segmentation_metric:   4.7605364<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.432310<br />segmentation_metric:   7.5481172<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 109.730126<br />segmentation_metric:   8.3992995<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 100.299750<br />segmentation_metric:   6.5525362<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  27.172020<br />segmentation_metric:   1.2698413<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  93.764211<br />segmentation_metric:   4.7319035<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 125.471581<br />segmentation_metric:   4.9303406<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  95.232011<br />segmentation_metric:   6.7536765<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  72.162102<br />segmentation_metric:   6.7784257<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.339494<br />segmentation_metric:   5.3262032<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  34.525826<br />segmentation_metric:   2.9055794<br />well_name: G09 <br>well_pos_x: 3 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  87.279325<br />segmentation_metric:   4.4620887<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  81.871533<br />segmentation_metric:   6.0384615<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 109.216776<br />segmentation_metric:   4.0198939<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 142.447656<br />segmentation_metric:  12.8629032<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.933610<br />segmentation_metric:   5.5543478<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  77.215704<br />segmentation_metric:   8.0572917<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  95.495918<br />segmentation_metric:   5.3111481<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.968993<br />segmentation_metric:   3.8015873<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 160.397344<br />segmentation_metric:   6.6781437<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  83.152007<br />segmentation_metric:   5.6790924<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 117.713725<br />segmentation_metric:   8.0429043<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  88.890239<br />segmentation_metric:   6.1856061<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 136.960985<br />segmentation_metric:   5.5447316<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  57.479043<br />segmentation_metric:   4.8217391<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 123.885451<br />segmentation_metric:   6.4627507<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  70.270647<br />segmentation_metric:   5.0980066<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  74.983942<br />segmentation_metric:   6.9596200<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 108.428547<br />segmentation_metric:   5.6975524<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length: 121.622740<br />segmentation_metric:   5.3952991<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  30.368613<br />segmentation_metric:   1.0000000<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  25.091238<br />segmentation_metric:   1.0681818<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6","Mean_Morphology_Major_Axis_Length:  17.208258<br />segmentation_metric:   2.2580645<br />well_name: G09 <br>well_pos_x: 4 <br>well_pos_y: 6"],"type":"scatter","mode":"markers","marker":{"autocolorscale":false,"color":"rgba(0,0,0,1)","opacity":1,"size":5.66929133858268,"symbol":"circle","line":{"width":1.88976377952756,"color":"rgba(0,0,0,1)"}},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-7.66303675,259.31076375],"y":[1.1,1.1],"text":"yintercept: 1.1","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null},{"x":[-7.66303675,259.31076375],"y":[13,13],"text":"yintercept: 13","type":"scatter","mode":"lines","line":{"width":1.88976377952756,"color":"rgba(255,0,0,1)","dash":"dash"},"hoveron":"points","showlegend":false,"xaxis":"x","yaxis":"y","hoverinfo":"text","frame":null}],"layout":{"margin":{"t":27.7601127123967,"r":7.30593607305936,"b":41.7144506119401,"l":43.1050228310502},"plot_bgcolor":"rgba(235,235,235,1)","paper_bgcolor":"rgba(255,255,255,1)","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187},"xaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-7.66303675,259.31076375],"tickmode":"array","ticktext":["0","50","100","150","200","250"],"tickvals":[0,50,100,150,200,250],"categoryorder":"array","categoryarray":["0","50","100","150","200","250"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"y","title":{"text":"Mean_Morphology_Major_Axis_Length","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"yaxis":{"domain":[0,1],"automargin":true,"type":"linear","autorange":false,"range":[-5.72786481253301,123.254121926891],"tickmode":"array","ticktext":["0","30","60","90","120"],"tickvals":[0,30,60,90,120],"categoryorder":"array","categoryarray":["0","30","60","90","120"],"nticks":null,"ticks":"outside","tickcolor":"rgba(51,51,51,1)","ticklen":3.65296803652968,"tickwidth":0.66417600664176,"showticklabels":true,"tickfont":{"color":"rgba(77,77,77,1)","family":"","size":11.689497716895},"tickangle":-0,"showline":false,"linecolor":null,"linewidth":0,"showgrid":true,"gridcolor":"rgba(255,255,255,1)","gridwidth":0.66417600664176,"zeroline":false,"anchor":"x","title":{"text":"segmentation_metric","font":{"color":"rgba(0,0,0,1)","family":"","size":14.6118721461187}},"hoverformat":".2f"},"shapes":[{"type":"rect","fillcolor":null,"line":{"color":null,"width":0,"linetype":[]},"yref":"paper","xref":"paper","x0":0,"x1":1,"y0":0,"y1":1}],"showlegend":false,"legend":{"bgcolor":"rgba(255,255,255,1)","bordercolor":"transparent","borderwidth":1.88976377952756,"font":{"color":"rgba(0,0,0,1)","family":"","size":11.689497716895}},"hovermode":"closest","barmode":"relative"},"config":{"doubleClick":"reset","showSendToCloud":false},"source":"A","attrs":{"19082ad3650e":{"x":{},"y":{},"text":{},"type":"scatter"},"1908630c17d5":{"yintercept":{}},"190834542323":{"yintercept":{}}},"cur_data":"19082ad3650e","visdat":{"19082ad3650e":["function (y) ","x"],"1908630c17d5":["function (y) ","x"],"190834542323":["function (y) ","x"]},"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-51e92edd0a4befec6a24">{"viewer":{"width":"100%","height":400,"padding":15,"fill":true},"browser":{"width":"100%","height":400,"padding":40,"fill":true}}</script>
</body>
</html>