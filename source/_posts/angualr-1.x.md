title: Angular 1.x 详解
date: 2016-10-13 20:35:00
description:
categories: 前端开发
tags:
  - Angular
toc:
feature:
---
## 前言
早前码了蛮久的Angular一直没能闲下来写点这方面的文章,Angular1.x以all in one特点加上脏检查的底层机制造成它过于臃肿与性能大瓶颈.前端最近五年,日新月异,而2015年React大热,Angular2.0在2016年9月正式发布了,与此同时Angular 1.x进入漫长的普及率下降通道,目前有些大型前端项目也很难平滑过度,还是写点文字记录一下.以下分析与此同时Angular1.4.9正式版代码为例.

### 启动过程
原源码接近3W行,整体启动过程:
* 1.检查angular.bootstrap
```
if (window.angular.bootstrap) {
  console.log('WARNING: Tried to load angular more than once.');
  return;
}
```
angular外面启动核心接口,angular.bootstrap保证引导启动angular项目且防止被多个实例重复引导启动.
* 2.绑定jqLite
```
bindJQuery();

angular.element = jqLite;//完成绑定
bindJQueryFired = true;//防止重绑
```
angular内部实现了精简版的jQ核心功能,如果jQuery优先于angular载入,则使用jQuery,否则使用其内部实现的jqLite.
* 3.载入angular所有外部接口
```
publishExternalAPI(angular);

function publishExternalAPI(angular) {
    extend(angular, {
        'bootstrap': bootstrap,
        //more
        //大量外部angular独立api
    });

    angularModule = setupModuleLoader(window);

    angularModule('ng', ['ngLocale'], ['$provide',
    function ngModule($provide) {
        $provide.provider({
            $$sanitizeUri: $$SanitizeUriProvider
        });
        $provide.provider('$compile', $CompileProvider).directive({
            a: htmlAnchorDirective,
            //more
            //大量官方内置指令集
        }).directive({
            ngInclude: ngIncludeFillContentDirective
        }).directive(ngAttributeAliasDirectives).directive(ngEventDirectives);
        $provide.provider({
            $anchorScroll: $AnchorScrollProvider,
            //more
            //
        });
    }]);
}
```
其中setupModuleLoader完成window.angular的全局注册,并返回绑定各种核心业务模型 如factory,provider,directive等等
```
function setupModuleLoader(window) {
    var $injectorMinErr = minErr('$injector');
    var ngMinErr = minErr('ng');
    //报错核心处理
    function ensure(obj, name, factory) {
        return obj[name] || (obj[name] = factory())
    }
    //防重载函数
    var angular = ensure(window, 'angular', Object);
    //核心注册window.angular
    angular.$$minErr = angular.$$minErr || minErr;
    return ensure(angular, 'module',
    function() {
        var modules = {};
        return function module(name, requires, configFn) {
            var assertNotHasOwnProperty = function(name, context) {
                if (name === 'hasOwnProperty') {
                    throw ngMinErr('badname', 'hasOwnProperty is not a valid {0} name', context);
                }
            };
            assertNotHasOwnProperty(name, 'module');
            if (requires && modules.hasOwnProperty(name)) {
                modules[name] = null;
            }
            return ensure(modules, name,
            function() {
                if (!requires) {
                    throw $injectorMinErr('nomod', "Module '{0}' is not available! You either misspelled the module name or forgot to load it. If registering a module ensure that you specify the dependencies as the second argument.", name);
                }
                var invokeQueue = [];
                var configBlocks = [];
                var runBlocks = [];
                var config = invokeLater('$injector', 'invoke', 'push', configBlocks);
                var moduleInstance = {
                    _invokeQueue: invokeQueue,
                    _configBlocks: configBlocks,
                    _runBlocks: runBlocks,
                    requires: requires,
                    name: name,
                    provider: invokeLaterAndSetModuleName('$provide', 'provider'),
                    factory: invokeLaterAndSetModuleName('$provide', 'factory'),
                    service: invokeLaterAndSetModuleName('$provide', 'service'),
                    value: invokeLater('$provide', 'value'),
                    constant: invokeLater('$provide', 'constant', 'unshift'),
                    decorator: invokeLaterAndSetModuleName('$provide', 'decorator'),
                    animation: invokeLaterAndSetModuleName('$animateProvider', 'register'),
                    filter: invokeLaterAndSetModuleName('$filterProvider', 'register'),
                    controller: invokeLaterAndSetModuleName('$controllerProvider', 'register'),
                    directive: invokeLaterAndSetModuleName('$compileProvider', 'directive'),
                    config: config,
                    run: function(block) {
                        runBlocks.push(block);
                        return this;
                    }
                };
                if (configFn) {
                    config(configFn);
                }
                return moduleInstance;
                function invokeLater(provider, method, insertMethod, queue) {
                    if (!queue) queue = invokeQueue;
                    return function() {
                        queue[insertMethod || 'push']([provider, method, arguments]);
                        return moduleInstance
                    }
                }
                function invokeLaterAndSetModuleName(provider, method) {
                    return function(recipeName, factoryFunction) {
                        if (factoryFunction && isFunction(factoryFunction)) factoryFunction.$$moduleName = name;
                        invokeQueue.push([provider, method, arguments]);
                        return moduleInstance
                    }
                }
            })
        }
    })
}
```
* 4.利用 angular.module 载入"ngLocale"核心内置模块,并启动设置angular国际化语言设置
```
angular.module("ngLocale", [], ["$provide",
function($provide) {
    //more code
    $provide.value("$locale", {
        //more code
    });
}]);
```
* 5.jqLite等待dom结构载入完成并启动angular初始化
```
jqLite(document).ready(function() {
    angularInit(document, bootstrap);
});
```
通过查找ng-app或者自定义前缀-app找到module名称,并启动angular.bootstrap引导,dom结构至上而下进行重新渲染整理.
```
function angularInit(element, bootstrap) {
    var appElement, module, config = {};
    forEach(
        ngAttrPrefixes,
        function(prefix) {
            var name = prefix + 'app';
            if (!appElement && element.hasAttribute && element.hasAttribute(name)) {
                appElement = element;
                module = element.getAttribute(name)
            }
        }
    );
    forEach(
        ngAttrPrefixes,
        function(prefix) {
            var name = prefix + 'app';
            var candidate;
            if (!appElement && (candidate = element.querySelector('[' + name.replace(':', '\\:') + ']'))) {
                appElement = candidate;
                module = candidate.getAttribute(name)
            }
        }
    );
    if (appElement) {
        config.strictDi = getNgAttribute(appElement, "strict-di") !== null;
        bootstrap(appElement, module ? [module] : [], config)
    }
}
```
至此整个angular完成初始化
```
function bootstrap(element, modules, config) {
    if (!isObject(config)) config = {};
    var defaultConfig = {
        strictDi: false
    };
    config = extend(defaultConfig, config);
    //用户app主模块启动核心
    var doBootstrap = function() {
        element = jqLite(element);
        if (element.injector()) {
            var tag = (element[0] === document) ? 'document': startingTag(element);
            throw ngMinErr('btstrpd', "App Already Bootstrapped with this Element '{0}'", tag.replace(/</, '&lt;').replace(/>/, '&gt;'));
        }
        modules = modules || [];
        modules.unshift([
            '$provide',
            function($provide) {
                $provide.value('$rootElement', element);
            }
        ]);
        if (config.debugInfoEnabled) {
            modules.push([
                '$compileProvider',
                function($compileProvider) {
                    $compileProvider.debugInfoEnabled(true);
                }
            ]);
        }
        modules.unshift('ng');
        var injector = createInjector(modules, config.strictDi);
        //至上而下逐层往dom的注入双向数据绑定,$rootScope主树scope开始;
        injector.invoke(['$rootScope', '$rootElement', '$compile', '$injector',
            function bootstrapApply(scope, element, compile, injector) {
                scope.$apply(function() {
                    element.data('$injector', injector);
                    compile(element)(scope);
                    //开始渲染
                });
            }
        ]);
        return injector;
    };
    var NG_ENABLE_DEBUG_INFO = /^NG_ENABLE_DEBUG_INFO!/;
    var NG_DEFER_BOOTSTRAP = /^NG_DEFER_BOOTSTRAP!/;
    if (window && NG_ENABLE_DEBUG_INFO.test(window.name)) {
        config.debugInfoEnabled = true;
        window.name = window.name.replace(NG_ENABLE_DEBUG_INFO, '');
    }
    if (window && !NG_DEFER_BOOTSTRAP.test(window.name)) {
        return doBootstrap();
    }
    window.name = window.name.replace(NG_DEFER_BOOTSTRAP, '');
    angular.resumeBootstrap = function(extraModules) {
        forEach(extraModules,
        function(module) {
            modules.push(module)
        });
        return doBootstrap()
    };
    if (isFunction(angular.resumeDeferredBootstrap)) {
        angular.resumeDeferredBootstrap()
    }
}
```
### 特性
#### 调用函数invoke
```
function invoke(fn, self, locals, serviceName) {
    if (typeof locals === 'string') {
        serviceName = locals;
        locals = null
    }
    var args = [],
    $inject = createInjector.$$annotate(fn, strictDi, serviceName),
    length,
    i,
    key;
    for (i = 0, length = $inject.length; i < length; i++) {
        key = $inject[i];
        if (typeof key !== 'string') {
            throw $injectorMinErr('itkn', 'Incorrect injection token! Expected service name as string, got {0}', key)
        }
        args.push(locals && locals.hasOwnProperty(key) ? locals[key] : getService(key, serviceName))
    }
    if (isArray(fn)) {
        fn = fn[length]
    }
    return fn.apply(self, args)
}
```
调用函数核心就是apply.
#### scope
```
function Scope() {
  this.$id = nextUid();
  this.$$phase = this.$parent = this.$$watchers =
                 this.$$nextSibling = this.$$prevSibling =
                 this.$$childHead = this.$$childTail = null;
  this.$root = this;
  this.$$destroyed = false;
  this.$$listeners = {};
  this.$$listenerCount = {};
  this.$$watchersCount = 0;
  this.$$isolateBindings = null;
}
```
基本scope核心属性包括监听相关,上下父子相邻级数据及每个闭包的$id,销毁相关.
1. watch实现核心
```
{
    $watchGroup: function(watchExpressions, listener) {
        var oldValues = new Array(watchExpressions.length);
        var newValues = new Array(watchExpressions.length);
        var deregisterFns = [];
        var self = this;
        var changeReactionScheduled = false;
        var firstRun = true;
        if (!watchExpressions.length) {
            var shouldCall = true;
            self.$evalAsync(function() {
                if (shouldCall) listener(newValues, newValues, self)
            });
            return function deregisterWatchGroup() {
                shouldCall = false
            }
        }
        if (watchExpressions.length === 1) {
            return this.$watch(watchExpressions[0],
            function watchGroupAction(value, oldValue, scope) {
                newValues[0] = value;
                oldValues[0] = oldValue;
                listener(newValues, (value === oldValue) ? newValues: oldValues, scope)
            })
        }
        forEach(watchExpressions,
        function(expr, i) {
            var unwatchFn = self.$watch(expr,
            function watchGroupSubAction(value, oldValue) {
                newValues[i] = value;
                oldValues[i] = oldValue;
                if (!changeReactionScheduled) {
                    changeReactionScheduled = true;
                    self.$evalAsync(watchGroupAction)
                }
            });
            deregisterFns.push(unwatchFn)
        });
        function watchGroupAction() {
            changeReactionScheduled = false;
            if (firstRun) {
                firstRun = false;
                listener(newValues, newValues, self)
            } else {
                listener(newValues, oldValues, self)
            }
        }
        return function deregisterWatchGroup() {
            while (deregisterFns.length) {
                deregisterFns.shift()()
            }
        }
    }
}
```
可以看到核心的$watchGroup以组数的存储各个版本的old与new,监听变量越多,次数越多内存占用越多,影响性能,因此$watch开销较大应该尽量少用.另外,$watchCollection是监听对象或者数组的,也是基于$watchGroup.每次计数器$$watchersCount.

2. $eval
```
{
    $eval: function(expr, locals) {
        return $parse(expr)(this, locals);
    }
}
```
$parse解析器基于$ParseProvider,其返回一个拦截器函数用于 输入应用同步数据.
```
function $ParseProvider(){
      var cacheDefault = createMap();
      var cacheExpensive = createMap();

      this.$get = ['$filter', function($filter) {
            var noUnsafeEval = csp().noUnsafeEval;
            var $parseOptions = {
                  csp: noUnsafeEval,
                  expensiveChecks: false
                },
                $parseOptionsExpensive = {
                  csp: noUnsafeEval,
                  expensiveChecks: true
                };
            return function $parse(){};
            function expressionInputDirtyCheck(){}
            function inputsWatchDelegate(){}
            function oneTimeWatchDelegate(){}
            function oneTimeLiteralWatchDelegate(){}
            function constantWatchDelegate(){}
            function addInterceptor(){}
      }];
}
```
$ParseProvider实现了最核心的解析器肮检查
```
function expressionInputDirtyCheck(newValue, oldValueOfValue) {
    if (newValue == null || oldValueOfValue == null) {
        return newValue === oldValueOfValue
    }
    if (typeof newValue === 'object') {
        newValue = getValueOf(newValue);
        if (typeof newValue === 'object') {
            return false
        }
    }
    return newValue === oldValueOfValue || (newValue !== newValue && oldValueOfValue !== oldValueOfValue)
}
```
expressionInputDirtyCheck实现了检查判断
```
function inputsWatchDelegate(scope, listener, objectEquality, parsedExpression, prettyPrintExpression) {
    var inputExpressions = parsedExpression.inputs;
    var lastResult;
    if (inputExpressions.length === 1) {
        var oldInputValueOf = expressionInputDirtyCheck;
        inputExpressions = inputExpressions[0];
        return scope.$watch(function expressionInputWatch(scope) {
            var newInputValue = inputExpressions(scope);
            if (!expressionInputDirtyCheck(newInputValue, oldInputValueOf)) {
                lastResult = parsedExpression(scope, undefined, undefined, [newInputValue]);
                oldInputValueOf = newInputValue && getValueOf(newInputValue)
            }
            return lastResult
        },
        listener, objectEquality, prettyPrintExpression)
    }
    var oldInputValueOfValues = [];
    var oldInputValues = [];
    for (var i = 0,
    ii = inputExpressions.length; i < ii; i++) {
        oldInputValueOfValues[i] = expressionInputDirtyCheck;
        oldInputValues[i] = null
    }
    return scope.$watch(function expressionInputsWatch(scope) {
        var changed = false;
        for (var i = 0,
        ii = inputExpressions.length; i < ii; i++) {
            var newInputValue = inputExpressions[i](scope);
            if (changed || (changed = !expressionInputDirtyCheck(newInputValue, oldInputValueOfValues[i]))) {
                oldInputValues[i] = newInputValue;
                oldInputValueOfValues[i] = newInputValue && getValueOf(newInputValue)
            }
        }
        if (changed) {
            lastResult = parsedExpression(scope, undefined, undefined, oldInputValues)
        }
        return lastResult
    },
    listener, objectEquality, prettyPrintExpression)
}
```
inputsWatchDelegate 完成输入脏检查后的scope更新.
3. $digest输出脏检查
基于$$watchers的postDigestQueue,产生队列并按队列更新
```
{
    $digest: function() {
        var watch, value, last, watchers, length, dirty, ttl = TTL,
        next, current, target = this,
        watchLog = [],
        logIdx,
        logMsg,
        asyncTask;
        beginPhase('$digest');
        $browser.$$checkUrlChange();
        if (this === $rootScope && applyAsyncId !== null) {
            $browser.defer.cancel(applyAsyncId);
            flushApplyAsync()
        }
        lastDirtyWatch = null;
        do {
            dirty = false;
            current = target;
            while (asyncQueue.length) {
                try {
                    asyncTask = asyncQueue.shift();
                    asyncTask.scope.$eval(asyncTask.expression, asyncTask.locals)
                } catch(e) {
                    $exceptionHandler(e)
                }
                lastDirtyWatch = null
            }
            traverseScopesLoop: do {
                if ((watchers = current.$$watchers)) {
                    length = watchers.length;
                    while (length--) {
                        try {
                            watch = watchers[length];
                            if (watch) {
                                if ((value = watch.get(current)) !== (last = watch.last) && !(watch.eq ? equals(value, last) : (typeof value === 'number' && typeof last === 'number' && isNaN(value) && isNaN(last)))) {
                                    dirty = true;
                                    lastDirtyWatch = watch;
                                    watch.last = watch.eq ? copy(value, null) : value;
                                    watch.fn(value, ((last === initWatchVal) ? value: last), current);
                                    if (ttl < 5) {
                                        logIdx = 4 - ttl;
                                        if (!watchLog[logIdx]) watchLog[logIdx] = [];
                                        watchLog[logIdx].push({
                                            msg: isFunction(watch.exp) ? 'fn: ' + (watch.exp.name || watch.exp.toString()) : watch.exp,
                                            newVal: value,
                                            oldVal: last
                                        })
                                    }
                                } else if (watch === lastDirtyWatch) {
                                    dirty = false;
                                    break traverseScopesLoop
                                }
                            }
                        } catch(e) {
                            $exceptionHandler(e)
                        }
                    }
                }
                if (! (next = ((current.$$watchersCount && current.$$childHead) || (current !== target && current.$$nextSibling)))) {
                    while (current !== target && !(next = current.$$nextSibling)) {
                        current = current.$parent
                    }
                }
            } while (( current = next ));
            if ((dirty || asyncQueue.length) && !(ttl--)) {
                clearPhase();
                throw $rootScopeMinErr('infdig', '{0} $digest() iterations reached. Aborting!\nWatchers fired in the last 5 iterations: {1}', TTL, watchLog)
            }
        } while ( dirty || asyncQueue . length );
        clearPhase();
        while (postDigestQueue.length) {
            try {
                postDigestQueue.shift()()
            } catch(e) {
                $exceptionHandler(e)
            }
        }
    }
}
```

核心检查
```
(
    value = watch.get(current)) !== (last = watch.last) && //当前值和最后记录的值不相等 初级检查
        !(
            watch.eq?
            equals(value, last)://深度检查
            (typeof value === 'number' && typeof last === 'number'&& isNaN(value) && isNaN(last))
        )
)
```
$apply事实上是 $eval解析,以及$digest'向前'的脏检查并渲染
```
{
    $apply: function(expr) {
        try {
            beginPhase('$apply');
            try {
                return this.$eval(expr);
            } finally {
                clearPhase();
            }
        } catch(e) {
            $exceptionHandler(e);
        } finally {
            try {
                $rootScope.$digest();
            } catch(e) {
                $exceptionHandler(e);
                throw e;
            }
        }
    }
}
```
#### 指令集 directive
```
this.directive = function registerDirective(name, directiveFactory) {
    assertNotHasOwnProperty(name, 'directive');
    if (isString(name)) {
        assertValidDirectiveName(name);
        assertArg(directiveFactory, 'directiveFactory');
        if (!hasDirectives.hasOwnProperty(name)) {
            hasDirectives[name] = [];
            $provide.factory(name + Suffix, ['$injector', '$exceptionHandler',
            function($injector, $exceptionHandler) {
                var directives = [];
                forEach(hasDirectives[name],
                function(directiveFactory, index) {
                    try {
                        var directive = $injector.invoke(directiveFactory);
                        if (isFunction(directive)) {
                            directive = {
                                compile: valueFn(directive)
                            }
                        } else if (!directive.compile && directive.link) {
                            directive.compile = valueFn(directive.link)
                        }
                        directive.priority = directive.priority || 0;
                        directive.index = index;
                        directive.name = directive.name || name;
                        directive.require = directive.require || (directive.controller && directive.name);
                        directive.restrict = directive.restrict || 'EA';
                        var bindings = directive.$$bindings = parseDirectiveBindings(directive, directive.name);
                        if (isObject(bindings.isolateScope)) {
                            directive.$$isolateBindings = bindings.isolateScope
                        }
                        directive.$$moduleName = directiveFactory.$$moduleName;
                        directives.push(directive)
                    } catch(e) {
                        $exceptionHandler(e)
                    }
                });
                return directives
            }])
        }
        hasDirectives[name].push(directiveFactory)
    } else {
        forEach(name, reverseParams(registerDirective))
    }
    return this
};
```
完整指令集核心渲染代码
```
function applyDirectivesToNode(directives, compileNode, templateAttrs, transcludeFn, jqCollection, originalReplaceDirective, preLinkFns, postLinkFns, previousCompileContext) {
    previousCompileContext = previousCompileContext || {};
    var terminalPriority = -Number.MAX_VALUE,
    newScopeDirective = previousCompileContext.newScopeDirective,
    controllerDirectives = previousCompileContext.controllerDirectives,
    newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective,
    templateDirective = previousCompileContext.templateDirective,
    nonTlbTranscludeDirective = previousCompileContext.nonTlbTranscludeDirective,
    hasTranscludeDirective = false,
    hasTemplate = false,
    hasElementTranscludeDirective = previousCompileContext.hasElementTranscludeDirective,
    $compileNode = templateAttrs.$$element = jqLite(compileNode),
    directive,
    directiveName,
    $template,
    replaceDirective = originalReplaceDirective,
    childTranscludeFn = transcludeFn,
    linkFn,
    directiveValue;
    for (var i = 0,
    ii = directives.length; i < ii; i++) {
        directive = directives[i];
        var attrStart = directive.$$start;
        var attrEnd = directive.$$end;
        if (attrStart) {
            $compileNode = groupScan(compileNode, attrStart, attrEnd)
        }
        $template = undefined;
        if (terminalPriority > directive.priority) {
            break
        }
        if (directiveValue = directive.scope) {
            if (!directive.templateUrl) {
                if (isObject(directiveValue)) {
                    assertNoDuplicate('new/isolated scope', newIsolateScopeDirective || newScopeDirective, directive, $compileNode);
                    newIsolateScopeDirective = directive
                } else {
                    assertNoDuplicate('new/isolated scope', newIsolateScopeDirective, directive, $compileNode)
                }
            }
            newScopeDirective = newScopeDirective || directive
        }
        directiveName = directive.name;
        if (!directive.templateUrl && directive.controller) {
            directiveValue = directive.controller;
            controllerDirectives = controllerDirectives || createMap();
            assertNoDuplicate("'" + directiveName + "' controller", controllerDirectives[directiveName], directive, $compileNode);
            controllerDirectives[directiveName] = directive
        }
        if (directiveValue = directive.transclude) {
            hasTranscludeDirective = true;
            if (!directive.$$tlb) {
                assertNoDuplicate('transclusion', nonTlbTranscludeDirective, directive, $compileNode);
                nonTlbTranscludeDirective = directive
            }
            if (directiveValue == 'element') {
                hasElementTranscludeDirective = true;
                terminalPriority = directive.priority;
                $template = $compileNode;
                $compileNode = templateAttrs.$$element = jqLite(document.createComment(' ' + directiveName + ': ' + templateAttrs[directiveName] + ' '));
                compileNode = $compileNode[0];
                replaceWith(jqCollection, sliceArgs($template), compileNode);
                childTranscludeFn = compile($template, transcludeFn, terminalPriority, replaceDirective && replaceDirective.name, {
                    nonTlbTranscludeDirective: nonTlbTranscludeDirective
                })
            } else {
                $template = jqLite(jqLiteClone(compileNode)).contents();
                $compileNode.empty();
                childTranscludeFn = compile($template, transcludeFn, undefined, undefined, {
                    needsNewScope: directive.$$isolateScope || directive.$$newScope
                })
            }
        }
        if (directive.template) {
            hasTemplate = true;
            assertNoDuplicate('template', templateDirective, directive, $compileNode);
            templateDirective = directive;
            directiveValue = (isFunction(directive.template)) ? directive.template($compileNode, templateAttrs) : directive.template;
            directiveValue = denormalizeTemplate(directiveValue);
            if (directive.replace) {
                replaceDirective = directive;
                if (jqLiteIsTextNode(directiveValue)) {
                    $template = []
                } else {
                    $template = removeComments(wrapTemplate(directive.templateNamespace, trim(directiveValue)))
                }
                compileNode = $template[0];
                if ($template.length != 1 || compileNode.nodeType !== NODE_TYPE_ELEMENT) {
                    throw $compileMinErr('tplrt', "Template for directive '{0}' must have exactly one root element. {1}", directiveName, '');
                }
                replaceWith(jqCollection, $compileNode, compileNode);
                var newTemplateAttrs = {
                    $attr: {}
                };
                var templateDirectives = collectDirectives(compileNode, [], newTemplateAttrs);
                var unprocessedDirectives = directives.splice(i + 1, directives.length - (i + 1));
                if (newIsolateScopeDirective || newScopeDirective) {
                    markDirectiveScope(templateDirectives, newIsolateScopeDirective, newScopeDirective);
                }
                directives = directives.concat(templateDirectives).concat(unprocessedDirectives);
                mergeTemplateAttributes(templateAttrs, newTemplateAttrs);
                ii = directives.length;
            } else {
                $compileNode.html(directiveValue);
            }
        }
        if (directive.templateUrl) {
            hasTemplate = true;
            assertNoDuplicate('template', templateDirective, directive, $compileNode);
            templateDirective = directive;
            if (directive.replace) {
                replaceDirective = directive;
            }
            nodeLinkFn = compileTemplateUrl(directives.splice(i, directives.length - i), $compileNode, templateAttrs, jqCollection, hasTranscludeDirective && childTranscludeFn, preLinkFns, postLinkFns, {
                controllerDirectives: controllerDirectives,
                newScopeDirective: (newScopeDirective !== directive) && newScopeDirective,
                newIsolateScopeDirective: newIsolateScopeDirective,
                templateDirective: templateDirective,
                nonTlbTranscludeDirective: nonTlbTranscludeDirective
            });
            ii = directives.length;
        } else if (directive.compile) {
            try {
                linkFn = directive.compile($compileNode, templateAttrs, childTranscludeFn);
                if (isFunction(linkFn)) {
                    addLinkFns(null, linkFn, attrStart, attrEnd);
                } else if (linkFn) {
                    addLinkFns(linkFn.pre, linkFn.post, attrStart, attrEnd);
                }
            } catch(e) {
                $exceptionHandler(e, startingTag($compileNode));
            }
        }
        if (directive.terminal) {
            nodeLinkFn.terminal = true;
            terminalPriority = Math.max(terminalPriority, directive.priority);
        }
    }
    nodeLinkFn.scope = newScopeDirective && newScopeDirective.scope === true;
    nodeLinkFn.transcludeOnThisElement = hasTranscludeDirective;
    nodeLinkFn.templateOnThisElement = hasTemplate;
    nodeLinkFn.transclude = childTranscludeFn;
    previousCompileContext.hasElementTranscludeDirective = hasElementTranscludeDirective;
    return nodeLinkFn;
    function addLinkFns(pre, post, attrStart, attrEnd) {
        if (pre) {
            if (attrStart) pre = groupElementsLinkFnWrapper(pre, attrStart, attrEnd);
            pre.require = directive.require;
            pre.directiveName = directiveName;
            if (newIsolateScopeDirective === directive || directive.$$isolateScope) {
                pre = cloneAndAnnotateFn(pre, {
                    isolateScope: true
                });
            }
            preLinkFns.push(pre);
        }
        if (post) {
            if (attrStart) post = groupElementsLinkFnWrapper(post, attrStart, attrEnd);
            post.require = directive.require;
            post.directiveName = directiveName;
            if (newIsolateScopeDirective === directive || directive.$$isolateScope) {
                post = cloneAndAnnotateFn(post, {
                    isolateScope: true
                });
            }
            postLinkFns.push(post);
        }
    }
    function getControllers(directiveName, require, $element, elementControllers) {
        var value;
        if (isString(require)) {
            var match = require.match(REQUIRE_PREFIX_REGEXP);
            var name = require.substring(match[0].length);
            var inheritType = match[1] || match[3];
            var optional = match[2] === '?';
            if (inheritType === '^^') {
                $element = $element.parent();
            } else {
                value = elementControllers && elementControllers[name];
                value = value && value.instance;
            }
            if (!value) {
                var dataName = '$' + name + 'Controller';
                value = inheritType ? $element.inheritedData(dataName) : $element.data(dataName);
            }
            if (!value && !optional) {
                throw $compileMinErr('ctreq', "Controller '{0}', required by directive '{1}', can't be found!", name, directiveName)
            }
        } else if (isArray(require)) {
            value = [];
            for (var i = 0,
            ii = require.length; i < ii; i++) {
                value[i] = getControllers(directiveName, require[i], $element, elementControllers)
            }
        }
        return value || null
    }
    function setupControllers($element, attrs, transcludeFn, controllerDirectives, isolateScope, scope) {
        var elementControllers = createMap();
        for (var controllerKey in controllerDirectives) {
            var directive = controllerDirectives[controllerKey];
            var locals = {
                $scope: directive === newIsolateScopeDirective || directive.$$isolateScope ? isolateScope: scope,
                $element: $element,
                $attrs: attrs,
                $transclude: transcludeFn
            };
            var controller = directive.controller;
            if (controller == '@') {
                controller = attrs[directive.name]
            }
            var controllerInstance = $controller(controller, locals, true, directive.controllerAs);
            elementControllers[directive.name] = controllerInstance;
            if (!hasElementTranscludeDirective) {
                $element.data('$' + directive.name + 'Controller', controllerInstance.instance)
            }
        }
        return elementControllers
    }
    function nodeLinkFn(childLinkFn, scope, linkNode, $rootElement, boundTranscludeFn) {
        var linkFn, isolateScope, controllerScope, elementControllers, transcludeFn, $element, attrs, removeScopeBindingWatches, removeControllerBindingWatches;
        if (compileNode === linkNode) {
            attrs = templateAttrs;
            $element = templateAttrs.$$element
        } else {
            $element = jqLite(linkNode);
            attrs = new Attributes($element, templateAttrs)
        }
        controllerScope = scope;
        if (newIsolateScopeDirective) {
            isolateScope = scope.$new(true)
        } else if (newScopeDirective) {
            controllerScope = scope.$parent
        }
        if (boundTranscludeFn) {
            transcludeFn = controllersBoundTransclude;
            transcludeFn.$$boundTransclude = boundTranscludeFn
        }
        if (controllerDirectives) {
            elementControllers = setupControllers($element, attrs, transcludeFn, controllerDirectives, isolateScope, scope)
        }
        if (newIsolateScopeDirective) {
            compile.$$addScopeInfo($element, isolateScope, true, !(templateDirective && (templateDirective === newIsolateScopeDirective || templateDirective === newIsolateScopeDirective.$$originalDirective)));
            compile.$$addScopeClass($element, true);
            isolateScope.$$isolateBindings = newIsolateScopeDirective.$$isolateBindings;
            removeScopeBindingWatches = initializeDirectiveBindings(scope, attrs, isolateScope, isolateScope.$$isolateBindings, newIsolateScopeDirective);
            if (removeScopeBindingWatches) {
                isolateScope.$on('$destroy', removeScopeBindingWatches)
            }
        }
        for (var name in elementControllers) {
            var controllerDirective = controllerDirectives[name];
            var controller = elementControllers[name];
            var bindings = controllerDirective.$$bindings.bindToController;
            if (controller.identifier && bindings) {
                removeControllerBindingWatches = initializeDirectiveBindings(controllerScope, attrs, controller.instance, bindings, controllerDirective)
            }
            var controllerResult = controller();
            if (controllerResult !== controller.instance) {
                controller.instance = controllerResult;
                $element.data('$' + controllerDirective.name + 'Controller', controllerResult);
                removeControllerBindingWatches && removeControllerBindingWatches();
                removeControllerBindingWatches = initializeDirectiveBindings(controllerScope, attrs, controller.instance, bindings, controllerDirective)
            }
        }
        for (i = 0, ii = preLinkFns.length; i < ii; i++) {
            linkFn = preLinkFns[i];
            invokeLinkFn(linkFn, linkFn.isolateScope ? isolateScope: scope, $element, attrs, linkFn.require && getControllers(linkFn.directiveName, linkFn.require, $element, elementControllers), transcludeFn)
        }
        var scopeToChild = scope;
        if (newIsolateScopeDirective && (newIsolateScopeDirective.template || newIsolateScopeDirective.templateUrl === null)) {
            scopeToChild = isolateScope
        }
        childLinkFn && childLinkFn(scopeToChild, linkNode.childNodes, undefined, boundTranscludeFn);
        for (i = postLinkFns.length - 1; i >= 0; i--) {
            linkFn = postLinkFns[i];
            invokeLinkFn(linkFn, linkFn.isolateScope ? isolateScope: scope, $element, attrs, linkFn.require && getControllers(linkFn.directiveName, linkFn.require, $element, elementControllers), transcludeFn)
        }
        function controllersBoundTransclude(scope, cloneAttachFn, futureParentElement) {
            var transcludeControllers;
            if (!isScope(scope)) {
                futureParentElement = cloneAttachFn;
                cloneAttachFn = scope;
                scope = undefined
            }
            if (hasElementTranscludeDirective) {
                transcludeControllers = elementControllers
            }
            if (!futureParentElement) {
                futureParentElement = hasElementTranscludeDirective ? $element.parent() : $element
            }
            return boundTranscludeFn(scope, cloneAttachFn, transcludeControllers, futureParentElement, scopeToChild)
        }
    }
}
```
compile与link不能共存,最后以factory形式存在
####  compile
* 至上而下 compileNodes然后渲染子节点,然后applyDirectivesToNode渲染指令directive
* service->factory->provider->providerCache 最后所有的区块存在providerCache中.
