
#编写可测试的Javascript
by REBECCA MURPHEY
原文地址： [writing-testable-javascript](http://alistapart.com/article/writing-testable-javascript "writing testable javascript")

我们都有过这样的经验：写几行Javscript代码来实现一个小功能，然后随着需求变化变成了十几行，然后二十几行，然后更多。在这个过程中，函数渐渐地接受更多的参数，条件判断渐渐地接受更多的条件。然后有一天，代码出了问题，我们开始从一堆杂乱的代码中找出bug，解决问题。

当我们要求客户端代码执行越来越多的功能时 --实际上，现在整个应用程序很大一部分都是在客户端实现的-- 有两件事情是可以确定的。1.我们无法通过敲敲键盘， 点点鼠标， 就确定功能实现无误， 而要通过自动化测试确保代码的正确性。2. 为了让代码便于自动化测试，或许我们需要稍微改变下写代码的方式。

虽然我们知道自动化测试很好，但是我们大部分现在都只能针对代码写集成测试。集成测试可以确保系统整体运行正常，但是无法帮我们确认单元功能的正常。

这就需要我们编写单元测试了。而编写单元测试会一直纠结直到我们开始*编写可测试的JavaScript*。

##单元VS集成： 区别在哪？

编写集成测试代码通常都非常直接：我们写代码来描述期待用户什么样的输入，以及用户应该期待她看见什么。
[Selenium](http://docs.seleniumhq.org/)是一个流行的自动化浏览器测试的工具。[Capybara](https://github.com/jnicklas/capybara)让在Ruby下使用Selenium非常容易。针对
许多其他的语言也有其专门的工具。

这里是针对一个搜索引擎的一部分的集成测试：

```
def test_search
  fill_in('q', :with => 'cat')
  find('.btn').click
  assert( find('#results li').has_content?('cat'), 'Search results are shown' )
  assert( page.has_no_selector?('#results li.no-results'), 'No results is not shown' )
end
```

与集成测试集中测试的是用户和程序的交互，单元测试仅仅关注小段代码的行为：
> 当我传递给一个函数一定的参数时，我有没有获得期望的返回值？

用传统方式写的代码可能会非常难写针对的单元测试用例-- 而且很难维护，调试，扩展。但是如果我们写代码时已经构思好了将要写的单元测试， 我们可能会发现不仅写单元测试变得很容易，我们的代码质量也在随之提高！

要知道为什么我这么说，我们先看个例子：

<img src="http://d.alistapart.com/375/app.png" alt="Srchr">

当用户输入一个搜索词时，程序会发送一个XHR请求到服务器来获取相应的数据。当服务器返回了JSON格式的数据后，客户端就会解析数据并且用客户端模板展示在页面上。用户可以点击一条搜索结果来表明他“喜欢(likes)”该结果，当该事件发生时，他“喜欢”的人名会被添加到右侧的喜欢列表上。

一个“传统”的JavaScript 实现方式可能是这个样子的：
```
var tmplCache = {};

function loadTemplate (name) {
  if (!tmplCache[name]) {
    tmplCache[name] = $.get('/templates/' + name);
  }
  return tmplCache[name];
}

$(function () {

  var resultsList = $('#results');
  var liked = $('#liked');
  var pending = false;

  $('#searchForm').on('submit', function (e) {
    e.preventDefault();

    if (pending) { return; }

    var form = $(this);
    var query = $.trim( form.find('input[name="q"]').val() );

    if (!query) { return; }

    pending = true;

    $.ajax('/data/search.json', {
      data : { q: query },
      dataType : 'json',
      success : function (data) {
        loadTemplate('people-detailed.tmpl').then(function (t) {
          var tmpl = _.template(t);
          resultsList.html( tmpl({ people : data.results }) );
          pending = false;
        });
      }
    });

    $('<li>', {
      'class' : 'pending',
      html : 'Searching &hellip;'
    }).appendTo( resultsList.empty() );
  });

  resultsList.on('click', '.like', function (e) {
    e.preventDefault();
    var name = $(this).closest('li').find('h2').text();
    liked.find('.no-results').remove();
    $('<li>', { text: name }).appendTo(liked);
  });

});

```

我的好友[ Adam Sontag ](https://twitter.com/ajpiano)称之为*随意冒险*代码--在任意一行代码上面，我们可能会执行展现，数据操作，交互 或者状态维护等任意操作。天知道！针对这种代码写集成测试很简单，但是单元功能却很难测试。

让单元测试变得困难的包括以下四点：

* 结构松散; 几乎全部操作都发生在 `$(document).ready()` 的回调函数中， 这是个匿名函数，没有关于它的接口提供测试。

* 复杂的函数; 如果一个函数就像提交处理函数一样超过10行，那么很可能它要执行的操作过多。

* 隐藏起来或者共享的程序状态： 例如 `pending ` 是在闭包中的变量，那就无法测试pending这个状态是不是正确的。

* 紧密的耦合; 例如 一个 `$.ajax `成功的回调函数不应该直接操作DOM。

##组织我们的代码
解决这个问题的第一步是让我们的代码更少的像面条一样扭在一起， 将其按功能分解到不同的区域：

*数据展现和用户交互
*数据管理和持久化
*程序状态
*设置和粘合代码来把所有代码整合在一起可以执行。

在上面的实现中，这四类操作交织在一起，在这行我们正在进行页面呈现， 两行之后就开始和服务器通信。

<img src="http://d.alistapart.com/375/code-lines.png" alt="Code Lines">

虽然我们可以为它写集成测试--我们也应该写--为这些代码写单元测试却非常困难。在功能性测试中，我们作出诸如“当用户搜索关键词，她应当看见正确的结果”的断言， 我们却无法更加具体。如果哪里出了问题，我们只能跟踪代码找到错误的地方，而功能性测试在这时候没什么作用。

如果我们重新思考如何编写代码， 那么， 我们就可以编写单元测试帮我们更好的找到出问题的地方， 而且也写出更容易复用，维护和扩展的代码。

我们的新代码会遵守以下原则

*将一组操作作为一个独立的对象，只执行四种功能之一，并且完全不知道其他对象的结构， 以此避免交织的代码。
*支持可配置性， 而非将东西都写死。以免我们为了测试而复制整个HTML环境
*保持对象代码的简洁，这有助于我们保持测试简单以及代码可读
*用构造函数来创建实例。这让我们可以为了测试的需求“干净地”复制每段代码。

作为开始，我们要先找到从哪里开始分解我们的代码。我们有三块用来做展现和交互： 搜索表单，搜索结果，以及“喜欢”框

<img src="http://d.alistapart.com/375/app-views.png" alt="Application Views">

同样会有一块代码用来从服务器取数据，以及另外一块用来将所有的代码放在一起。

让我们先看看代码中最简单的一块： Like框。在初始的版本中，这些代码用来更新Like框： 

```
var liked = $('#liked');

var resultsList = $('#results');


// ...


resultsList.on('click', '.like', function (e) {
  e.preventDefault();

  var name = $(this).closest('li').find('h2').text();

  liked.find( '.no-results' ).remove();

  $('<li>', { text: name }).appendTo(liked);

});

```

搜索结果块完全和Like框的代码纠缠在了一起，需要知道它的很多的标签结构。 一个更好的， 更易测试的方法是创建一个Likes对象用来执行和Like框相关的DOM操作：

```
var Likes = function (el) {
  this.el = $(el);
  return this;
};

Likes.prototype.add = function (name) {
  this.el.find('.no-results').remove();
  $('<li>', { text: name }).appendTo(this.el);
};
```

这段代码提供了一个用来创建新Like框实例的构造函数。这个实例会有个.add()方法用来添加搜索结果。这样我们就可以为它写单元测试了。

```

var ul;

setup(function(){
  ul = $('<ul><li class="no-results"></li></ul>');
});

test('constructor', function () {
  var l = new Likes(ul);
  assert(l);
});

test('adding a name', function () {
  var l = new Likes(ul);
  l.add('Brendan Eich');

  assert.equal(ul.find('li').length, 1);
  assert.equal(ul.find('li').first().html(), 'Brendan Eich');
  assert.equal(ul.find('li.no-results').length, 0);
});

```

没有那么难，是吗？ 这里我们使用[ Mocha](http://visionmedia.github.io/mocha/)作为测试框架， [Chai](http://chaijs.com/)作为测试库。 Mocha提供了`test`和`setup`函数; Chai提供了assert函数。还有许多其他可选的测试框架和库， 但这里为了说明作用， 这两个库足够了。在实践中可以选择最适合自己和项目的: 除了Mocha， [Qunit](http://qunitjs.com/)很流行， [Intern]是一个很有前景的框架。

我们的测试代码先创建了一个用来作为Like框容器的ul。 然后执行了两个测试： 一个用来测试我们是否成功创建了Like框，另一个用来测试 `.add()` 的正确性。 有了这些测试，我们就可以放心地重构Like框相关的代码， 并且知道有没有对现有功能造成破坏。

现在我们的客户端代码可以改成这样： 

```
var liked = new Likes('#liked');
var resultsList = $('#results');



// ...



resultsList.on('click', '.like', function (e) {
  e.preventDefault();

  var name = $(this).closest('li').find('h2').text();

  liked.add(name);
});
```

搜索结果模块要比Like框复杂， 让我们也试着重构一下。 就像我们在Like框上添加了个`add()`方法一样，我们需要用来和搜索结果进行交互的方法。 我们想要添加新结果的方法，





















