# 자바스크립트 연동

`노트: Blazor는 아직 기술 지원이 제공되지 않는 실험용 웹 프레임워크로, 실무 개발에 사용되어서는 안됩니다.`

A Blazor app can invoke JavaScript functions from .NET and .NET methods from JavaScript code.

### Invoke JavaScript functions from .NET methods <a id="invoke-javascript-functions-from-net-methods"></a>

There are times when Blazor .NET code is required to call a JavaScript function. For example, a JavaScript call can expose browser capabilities or functionality from a JavaScript library to the Blazor app.

To call into JavaScript from .NET, use the `IJSRuntime` abstraction, which is accessible from `JSRuntime.Current`. The `InvokeAsync<T>` method on `IJSRuntime` takes an identifier for the JavaScript function you wish to invoke along with any number of JSON-serializable arguments. The function identifier is relative to the global scope \(`window`\). If you wish to call `window.someScope.someFunction`, the identifier is `someScope.someFunction`. There's no need to register the function before it's called. The return type `T` must also be JSON serializable.

In the sample app, two JavaScript functions are available to the client-side app that interact with the DOM to receive user input and display a welcome message:

* `showPrompt` – Produces a prompt to accept user input \(the user's name\) and returns the name to the caller.
* `displayWelcome` – Assigns a welcome message from the caller to a DOM object with an `id` of `welcome`.

_wwwroot/exampleJsInterop.js_ 는 다음과 같습니다.

```text
window.exampleJsFunctions = {
  showPrompt: function (text) {
    return prompt(text, 'Type your name here');
  },
  displayWelcome: function (welcomeMessage) {
    document.getElementById('welcome').innerText = welcomeMessage;
  },
    returnArrayAsyncJs: function () {
      DotNet.invokeMethodAsync('BlazorSample', 'ReturnArrayAsync')
        .then(data => {
          data.push(4);
            console.log(data);
    })
  },
  sayHello: function (dotnetHelper) {
    return dotnetHelper.invokeMethodAsync('SayHello')
      .then(r => console.log(r));
  }
};
```

다음과 같이 자바스크립트 파일을 참조하는 `<script>` 태그를 _wwwroot/index.html_ 파일에 추가합니다.

```text
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width">
    <title>BlazorSample</title>
    <base href="/" />
    <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet">
    <link href="css/site.css" rel="stylesheet">
</head>
<body>
    <app>Loading...</app>

    <script src="css/bootstrap/bootstrap-native.min.js"></script>
    <script src="_framework/blazor.webassembly.js"></script>
    <script src="exampleJsInterop.js"></script>
</body>
</html>
```

스크립트 태그는 동적으로 업데이트 되지 않으므로 컴포넌트 파일에 스크립트 태그를 추가하지 마십시오.

.NET 메서드에서는 `IJSRuntime`의 `InvokeAsync<T>`메소드를 호출하여 자바스크립트 함수를 연동합니다.

예제 앱에서는 두 개의 C\# 메소드, `Prompt`와 `Display`를 사용하여 `showPrompt`와 `displayWelcome` 자바스크립트 함수를 다음과 같이 호출합니다.

_JsInteropClasses/ExampleJsInterop.cs_ 는 다음과 같습니다.

```text
public class ExampleJsInterop
{
    public static Task<string> Prompt(string text)
    {
        // showPrompt는 wwwroot/exampleJsInterop.js에 구현됩니다.
        return JSRuntime.Current.InvokeAsync<string>(
            "exampleJsFunctions.showPrompt",
            text);
    }

    public static Task<string> Display(string welcomeMessage)
    {
        // displayWelcome는 wwwroot/exampleJsInterop.js에 구현됩니다.
        return JSRuntime.Current.InvokeAsync<string>(
            "exampleJsFunctions.displayWelcome",
            welcomeMessage);
    }
    
    public static Task CallHelloHelperSayHello(string name)
    {
        // sayHello는 wwwroot/exampleJsInterop.js에 구현됩니다.
        return JSRuntime.Current.InvokeAsync<object>(
            "exampleJsFunctions.sayHello",
            new DotNetObjectRef(new HelloHelper(name)));
    }
}
```

The `IJSRuntime` abstraction is asynchronous to allow for server-side scenarios. If the app runs client-side and you want to invoke a JavaScript function synchronously, downcast to `IJSInProcessRuntime` and call `Invoke<T>` instead. We recommend that most JavaScript interop libraries use the async APIs to ensure the libraries are available in all Blazor scenarios, client-side or server-side.

The sample app includes a component to demonstrate JS interop. The component:

* Receives user input via a JS prompt.
* Returns the text to the component for processing.
* Calls a second JS function that interacts with the DOM to display a welcome message.

_Pages/JSInterop.cshtml_:

```text
@page "/JSInterop"
@using BlazorSample.JsInteropClasses
@using Microsoft.JSInterop;

<h1>JavaScript Interop</h1>

<h2>Invoke JavaScript functions from .NET methods</h2>

<button type="button" class="btn btn-primary" onclick="@TriggerJsPrompt">
    Trigger JavaScript Prompt
</button>

<h3 id="welcome" style="color:green;font-style:italic"></h3>

@functions {
    public async void TriggerJsPrompt()
    {
        var name = await ExampleJsInterop.Prompt("What's your name?");
        await ExampleJsInterop.Display($"Hello {name}! Welcome to Blazor!");
    }
}
```

1. When `TriggerJsPrompt` is executed by selecting the component's **Trigger JavaScript Prompt** button, the `ExampleJsInterop.Prompt` method in C\# code is called.
2. The `Prompt` method executes the JavaScript `showPrompt` function provided in the _wwwroot/exampleJsInterop.js_ file.
3. The `showPrompt` function accepts user input \(the user's name\), which is HTML-encoded and returned to the `Prompt` method and ultimately back to the component. The component stores the user's name in a local variable, `name`.
4. The string stored in `name` is incorporated into a welcome message, which is passed to a second C\# method, `ExampleJsInterop.Display`.
5. `Display` calls a JavaScript function, `displayWelcome`, which renders the welcome message into a heading tag.

### Capture references to elements <a id="capture-references-to-elements"></a>

Some [JavaScript interop](https://blazor.net/docs/javascript-interop.html) scenarios require references to HTML elements. For example, a UI library may require an element reference for initialization, or you might need to call command-like APIs on an element, such as `focus` or `play`.

You can capture references to HTML elements in a component by adding a `ref` attribute to the HTML element and then defining a field of type `ElementRef` whose name matches the value of the `ref` attribute.

The following example shows capturing a reference to the username input element:

```text
<input ref="username" ... />

@functions {
    ElementRef username;
}
```

**NOTE**

Do **not** use captured element references as a way of populating the DOM. Doing so may interfere with Blazor's declarative rendering model.

As far as .NET code is concerned, an `ElementRef` is an opaque handle. The _only_ thing you can do with it is pass it through to JavaScript code via JavaScript interop. When you do so, the JavaScript-side code receives an `HTMLElement` instance, which it can use with normal DOM APIs.

For example, the following code defines a .NET extension method that enables setting the focus on an element:

_mylib.js_:

```text
window.myLib = {
  focusElement : function (element) {
    element.focus();
  }
}
```

_ElementRefExtensions.cs_:

```text
using Microsoft.AspNetCore.Blazor;
using Microsoft.JSInterop;
using System.Threading.Tasks;

namespace MyLib
{
    public static class MyLibElementRefExtensions
    {
        public static Task Focus(this ElementRef elementRef)
        {
            return JSRuntime.Current.InvokeAsync<object>("myLib.focusElement", elementRef);
        }
    }
}
```

Now you can focus inputs in any of your components:

```text
@using MyLib

<input ref="username" />
<button onclick="@SetFocus">Set focus</button>

@functions {
    ElementRef username;

    void SetFocus()
    {
        username.Focus();
    }
}
```

**IMPORTANT**

The `username` variable is only populated after the component renders and its output includes the `<input>` element. If you try to pass an unpopulated `ElementRef` to JavaScript code, the JavaScript code receives `null`. To manipulate element references after the component has finished rendering \(to set the initial focus on an element\) use the `OnAfterRenderAsync` or `OnAfterRender` [component lifecycle methods](https://blazor.net/docs/components/index.html#lifecycle-methods).

### Invoke .NET methods from JavaScript functions <a id="invoke-net-methods-from-javascript-functions"></a>

#### Static .NET method call <a id="static-net-method-call"></a>

To invoke a static .NET method from JavaScript, use the `DotNet.invokeMethod` or `DotNet.invokeMethodAsync` functions. Pass in the identifier of the static method you wish to call, the name of the assembly containing the function, and any arguments. Again, the async version is preferred to support server-side scenarios. To be invokable from JavaScript, the .NET method must be public, static, and decorated with `[JSInvokable]`. By default, the method identifier is the method name, but you can specify a different identifier using the `JSInvokableAttribute`constructor. Calling open generic methods isn't currently supported.

The sample app includes a C\# method to return an array of `int`s. The method is decorated with the `JSInvokable` attribute.

_Pages/JsInterop.cshtml_:

```text
<button type="button" class="btn btn-primary"
        onclick="exampleJsFunctions.returnArrayAsyncJs()">
    Trigger .NET static method ReturnArrayAsync
</button>

@functions {
    [JSInvokable]
    public static Task<int[]> ReturnArrayAsync()
    {
        return Task.FromResult(new int[] { 1, 2, 3 });
    }
}
```

JavaScript served to the client invokes the C\# .NET method.

_wwwroot/exampleJsInterop.js_:

```text
window.exampleJsFunctions = {
  showPrompt: function (text) {
    return prompt(text, 'Type your name here');
  },
  displayWelcome: function (welcomeMessage) {
    document.getElementById('welcome').innerText = welcomeMessage;
  },
    returnArrayAsyncJs: function () {
      DotNet.invokeMethodAsync('BlazorSample', 'ReturnArrayAsync')
        .then(data => {
          data.push(4);
            console.log(data);
    })
  },
  sayHello: function (dotnetHelper) {
    return dotnetHelper.invokeMethodAsync('SayHello')
      .then(r => console.log(r));
  }
};
```

When the **Trigger .NET static method ReturnArrayAsync** button is selected, examine the console output in the browser's web developer tools:

```text
Array(4) [ 1, 2, 3, 4 ]
```

The fourth array value is pushed to the array \(`data.push(4);`\) returned by `ReturnArrayAsync`.

#### Instance method call <a id="instance-method-call"></a>

You can also call .NET instance methods from JavaScript. To invoke a .NET instance method from JavaScript, first pass the .NET instance to JavaScript by wrapping it in a `DotNetObjectRef`instance. The .NET instance is passed by reference to JavaScript, and you can invoke .NET instance methods on the instance using the `invokeMethod` or `invokeMethodAsync` functions. The .NET instance can also be passed as an argument when invoking other .NET methods from JavaScript.

**NOTE**

The sample app logs messages to the client-side console. For the following examples demonstrated by the sample app, examine the browser's console output in the browser's developer tools.

When the **Trigger .NET instance method HelloHelper.SayHello** button is selected, `ExampleJsInterop.CallHelloHelperSayHello` is called and passes a name, `Blazor`, to the method.

_Pages/JsInterop.cshtml_:

```text
<button type="button" class="btn btn-primary" onclick="@TriggerNetInstanceMethod">
    Trigger .NET instance method HelloHelper.SayHello
</button>

@functions {
    public async void TriggerNetInstanceMethod()
    {
        await ExampleJsInterop.CallHelloHelperSayHello("Blazor");
    }
}
```

`CallHelloHelperSayHello` invokes the JavaScript function `sayHello` with a new instance of `HelloHelper`.

_JsInteropClasses/ExampleJsInterop.cs_:

```text
public class ExampleJsInterop
{
    public static Task<string> Prompt(string text)
    {
        // showPrompt is implemented in wwwroot/exampleJsInterop.js
        return JSRuntime.Current.InvokeAsync<string>(
            "exampleJsFunctions.showPrompt",
            text);
    }

    public static Task<string> Display(string welcomeMessage)
    {
        // displayWelcome is implemented in wwwroot/exampleJsInterop.js
        return JSRuntime.Current.InvokeAsync<string>(
            "exampleJsFunctions.displayWelcome",
            welcomeMessage);
    }
    
    public static Task CallHelloHelperSayHello(string name)
    {
        // sayHello is implemented in wwwroot/exampleJsInterop.js
        return JSRuntime.Current.InvokeAsync<object>(
            "exampleJsFunctions.sayHello",
            new DotNetObjectRef(new HelloHelper(name)));
    }
}
```

_wwwroot/exampleJsInterop.js_:

```text
window.exampleJsFunctions = {
  showPrompt: function (text) {
    return prompt(text, 'Type your name here');
  },
  displayWelcome: function (welcomeMessage) {
    document.getElementById('welcome').innerText = welcomeMessage;
  },
    returnArrayAsyncJs: function () {
      DotNet.invokeMethodAsync('BlazorSample', 'ReturnArrayAsync')
        .then(data => {
          data.push(4);
            console.log(data);
    })
  },
  sayHello: function (dotnetHelper) {
    return dotnetHelper.invokeMethodAsync('SayHello')
      .then(r => console.log(r));
  }
};
```

The name is passed to `HelloHelper`'s constructor, which sets the `HelloHelper.Name` property. When the JavaScript function `sayHello` is executed, `HelloHelper.SayHello` returns the `Hello, {Name}!` message, which is written to the console by the JavaScript function.

_JsInteropClasses/HelloHelper.cs_:

```text
public class HelloHelper
{
    public HelloHelper(string name)
    {
        Name = name;
    }

    public string Name { get; set; }

    [JSInvokable]
    public string SayHello() => $"Hello, {Name}!";
}
```

Console output in the browser's web developer tools:

```text
Hello, Blazor!
```

### Share interop code in a Blazor class library <a id="share-interop-code-in-a-blazor-class-library"></a>

JavaScript interop code can be included in a Blazor class library \(`dotnet new blazorlib`\), which allows you to share the code in a NuGet package.

The Blazor class library handles embedding JavaScript resources in the built assembly. The JavaScript files are placed in the _wwwroot_ folder, and the tooling takes care of embedding the resources when the library is built.

The built NuGet package is referenced in the project file of a Blazor app just as any normal NuGet package is referenced. After the app has been restored, app code can call into JavaScript as if it were C\#.

