
#编写可测试的Javascript
by REBECCA MURPHEY
原文地址： [writing-testable-javascript](http://alistapart.com/article/writing-testable-javascript "writing testable javascript")

我们都有过这样的经验：写几行Javscript代码来实现一个小功能，然后随着需求变化变成了十几行，然后二十几行，然后更多。在这个过程中，函数渐渐地接受更多的参数，条件判断渐渐地接受更多的条件。然后有一天，代码出了问题，我们开始从一堆杂乱的代码中找出bug，解决问题。

当我们要求客户端代码执行越来越多的功能时 --实际上，现在整个应用程序很大一部分都是在客户端实现的-- 有两件事情是可以确定的。1.我们无法通过敲敲键盘， 点点鼠标， 就确定功能实现无误， 而要通过自动化测试确保代码的正确性。2. 为了让代码便于自动化测试，或许我们需要稍微改变下写代码的方式。

虽然我们知道自动化测试很好，但是我们大部分现在都只能针对代码写集成测试。集成测试可以确保系统整体运行正常，但是无法帮我们确认单元功能的正常。

这就需要我们编写单元测试了。而编写单元测试会一直纠结直到我们开始*编写可测试的JavaScript*。

##单元VS集成： 区别在哪？

编写集成测试代码通常都非常直接：我们写代码来描述期待用户什么样的输入，以及用户应该期待她看见什么。
[Selenium](http://docs.seleniumhq.org/)是一个流行的自动化浏览器测试工具。[Capybara](https://github.com/jnicklas/capybara)让在Ruby下使用Selenium非常容易。针对许多其他的语言也有其专门的工具。

这里是针对一个搜索引擎一个模块的集成测试：

```
def test_search
  fill_in('q', :with => 'cat')
  find('.btn').click
  assert( find('#results li').has_content?('cat'), 'Search results are shown' )
  assert( page.has_no_selector?('#results li.no-results'), 'No results is not shown' )
end
```

与集成测试关注的是用户和程序的交互，单元测试仅仅关注小段代码的行为：
> 当我传递给一个函数一定的参数时，我有没有获得期望的返回值？

用传统方式写的代码可能会非常难写对应的单元测试用例-- 而且很难维护，调试，扩展。但是如果我们写代码时已经构思好了将要写的单元测试， 我们可能会发现不仅编写单元测试变得很容易，我们的代码质量也会随之提高！

要知道为什么我这么说，我们先看个例子：

<img src="http://d.alistapart.com/375/app.png" alt="Srchr">

当用户输入一个搜索词时，程序会发送一个XHR请求到服务器获取相应数据。服务器返回了JSON格式的数据后，客户端就会解析数据并且用模板展示在页面上。用户可以点击一条搜索结果来表明他“喜欢(likes)”该结果，当该事件发生时，他Like的人名会被添加到右侧的Like列表上。

一个“传统”的JavaScript实现方式可能是这个样子的：
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

我的好友[ Adam Sontag ](https://twitter.com/ajpiano)称之为*任意冒险*代码--在任意一行代码我们可能会执行展现，数据操作，交互 或者状态维护等任意操作。天知道！针对这种代码写集成测试很简单，但是单元功能难以测试。

让单元测试变得困难的包括以下四点：

* 结构松散； 几乎全部操作都发生在 `$(document).ready()` 的回调函数中， 这是个匿名函数，没有接口可供测试。
* 复杂的函数； 如果一个函数超过10行--如同一个提交处理函数--那么很可能它要执行的操作过多。
* 隐藏起来或者共享的程序状态； 例如 `pending ` 是在闭包中的变量，那就无法测试pending这个状态是不是正确的。
* 紧密的耦合； 例如 一个 `$.ajax ` 成功（success）的回调函数不应该直接操作DOM。

##组织代码

解决这个问题的第一步是让我们的代码更少的像面条一样扭在一起， 将其按功能分解到不同的区域：

* 数据展现和用户交互
* 数据管理和持久化
* 程序状态
* 设置和粘合代码来把所有代码整合在一起

在上面的实现中，这四类操作交织在一起，在这行我们正在进行页面呈现， 两行之后就开始和服务器通信。

<img src="http://d.alistapart.com/375/code-lines.png" alt="Code Lines">

虽然我们可以写集成测试--我们也应该这么做--为这些代码写单元测试却非常困难。在功能性测试中，我们作出诸如“当用户搜索关键词，她应当看见正确的结果”的断言，却无法更加具体。如果哪里出了问题，就只能跟踪代码找到错误的地方，而功能性测试在这时候没什么作用。

重新思考如何编写代码可以帮我们编写单元测试以更好的找到出bug的地方， 而且这样写出的代码也更容易复用，维护和扩展。

我们的新代码会遵守以下原则

* 将一组操作作为一个独立的对象，只执行四种功能之一，并且一个对象完全不知道其他对象的结构， 以此避免交织的代码。
* 支持可配置性， 而非将东西都写死。以免我们为了测试而复制整个HTML环境
* 保持对象代码的简洁，这有助于我们保持测试简单以及代码可读
* 用构造函数来创建实例。这让我们可以为了测试的需求“干净地”复制每段代码。

作为开始，我们要先找到从哪里开始分解代码。有三块用来做展现和交互： 搜索表单，搜索结果，以及“Like框”

<img src="http://d.alistapart.com/375/app-views.png" alt="Application Views">

同样会有一块代码用来从服务器取数据，以及另外一块用来将其他模块组织在一起。

让我们先看看代码中最简单的： Like框。在初始的版本中，这些代码用来更新Like框： 

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

搜索结果模块完全和Like框的代码纠缠在了一起，需要知道它的标签结构信息。 一个更优雅的， 更易测试的方法是创建一个Likes对象用来执行和Like框相关的DOM操作：

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

这段代码提供了一个用来创建新Like框实例的构造函数。这个实例会有个`.add()`方法用来添加搜索结果。这样我们就可以为它写单元测试了。

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

没有那么难，是吗？ 这里我们使用[Mocha](http://visionmedia.github.io/mocha/)作为测试框架， [Chai](http://chaijs.com/)作为测试库。 Mocha提供了`test`和`setup`函数; Chai提供了assert函数。还有许多其他可选的测试框架和库， 但这里为了说明作用， 这两个库足够了。在实践中可以选择最适合自己和项目的: 除了Mocha， [Qunit](http://qunitjs.com/)很流行， [Intern](http://theintern.io/)显示了很多潜力。

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

搜索结果模块要比Like框复杂， 但我们也可以尝试重构。 如同我们在Like框上添加了个`add()`方法一样，我们需要用来和搜索结果进行交互的方法。 我们想要一个方法用来添加新结果，还有一个方法可以通知程序其他部分搜索结果发生了什么--例如，有人Like了一个搜索结果。

```
var SearchResults = function (el) {
  this.el = $(el);
  this.el.on( 'click', '.btn.like', _.bind(this._handleClick, this) );
};

SearchResults.prototype.setResults = function (results) {
  var templateRequest = $.get('people-detailed.tmpl');
  templateRequest.then( _.bind(this._populate, this, results) );
};

SearchResults.prototype._handleClick = function (evt) {
  var name = $(evt.target).closest('li.result').attr('data-name');
  $(document).trigger('like', [ name ]);
};

SearchResults.prototype._populate = function (results, tmpl) {
  var html = _.template(tmpl, { people: results });
  this.el.html(html);
};
```

现在， 我们原来的代码就可以改成： 

```
var liked = new Likes('#liked');
var resultsList = new SearchResults('#results');


// ...


$(document).on('like', function (evt, name) {
  liked.add(name);
})

```

这要比原来的实现简洁多了， 因为我们将`document`作为一个传递消息的枢纽，组件之间就可以在不知道彼此结构的情况下进行通信。（在一个商业的APP中，我们一般会使用诸如[Backbone](http://backbonejs.org/)或者是[RSVP](https://github.com/tildeio/rsvp.js)这样的框架来管理事件，这里我们用document来确保简洁。）我们也将复杂的操作--例如从被喜欢的搜索结果中找出人名--交给SearchResults对象，而非让它把代码结构整的乱七八糟。最好的是：现在我们同样可以为SearchResults编写单元测试了：

```
var ul;
var data = [ /* fake data here */ ];

setup(function () {
  ul = $('<ul><li class="no-results"></li></ul>');
});

test('constructor', function () {
  var sr = new SearchResults(ul);
  assert(sr);
});

test('display received results', function () {
  var sr = new SearchResults(ul);
  sr.setResults(data);

  assert.equal(ul.find('.no-results').length, 0);
  assert.equal(ul.find('li.result').length, data.length);
  assert.equal(
    ul.find('li.result').first().attr('data-name'),
    data[0].name
  );
});

test('announce likes', function() {
  var sr = new SearchResults(ul);
  var flag;
  var spy = function () {
    flag = [].slice.call(arguments);
  };

  sr.setResults(data);
  $(document).on('like', spy);

  ul.find('li').first().find('.like.btn').click();

  assert(flag, 'event handler called');
  assert.equal(flag[1], data[0].name, 'event handler receives data' );
});

```

和服务器的通信怎样模块化同样值得深思。原来是直接调用一个`$.ajax()`请求, 而且回调函数也在直接对DOM进行操作：

```
$.ajax('/data/search.json', {
  data : { q: query },
  dataType : 'json',
  success : function( data ) {
    loadTemplate('people-detailed.tmpl').then(function(t) {
      var tmpl = _.template( t );
      resultsList.html( tmpl({ people : data.results }) );
      pending = false;
    });
  }
});
```

同样地， 很难为这块代码编写单元测试， 短短的几行执行了太多的操作。我们将数据部分重构为一个对象：

```
var SearchData = function () { };

SearchData.prototype.fetch = function (query) {
  var dfd;

  if (!query) {
    dfd = $.Deferred();
    dfd.resolve([]);
    return dfd.promise();
  }

  return $.ajax( '/data/search.json', {
    data : { q: query },
    dataType : 'json'
  }).pipe(function( resp ) {
    return resp.results;
  });
};
```

现在， 可以修改获取数据部分的实现了：
```
var resultsList = new SearchResults('#results');

var searchData = new SearchData();

// ...

searchData.fetch(query).then(resultsList.setResults);
```

我们再一次对代码成功进行了简化， 并且将数据操作的复杂性封装在了SearchData对象里， 而非放在主程序中。
现在可以测试搜索接口了， 尽管在测试时有一些需要注意的地方：

* 我们不想直接和服务器进行交互--否则就要重新进行集成测试， 而且我们都是负责任的程序员， 所以服务器的代码肯定被测试过了，不是吗？ 相应地， 我们“模仿”和服务器的交互，通过[Sinon](http://sinonjs.org/)。
* 我们同样要测试一些极端情况， 例如空字符串搜索。

```
test('constructor', function () {
  var sd = new SearchData();
  assert(sd);
});

suite('fetch', function () {
  var xhr, requests;

  setup(function () {
    requests = [];
    xhr = sinon.useFakeXMLHttpRequest();
    xhr.onCreate = function (req) {
      requests.push(req);
    };
  });

  teardown(function () {
    xhr.restore();
  });

  test('fetches from correct URL', function () {
    var sd = new SearchData();
    sd.fetch('cat');

    assert.equal(requests[0].url, '/data/search.json?q=cat');
  });

  test('returns a promise', function () {
    var sd = new SearchData();
    var req = sd.fetch('cat');

    assert.isFunction(req.then);
  });

  test('no request if no query', function () {
    var sd = new SearchData();
    var req = sd.fetch();
    assert.equal(requests.length, 0);
  });

  test('return a promise even if no query', function () {
    var sd = new SearchData();
    var req = sd.fetch();

    assert.isFunction( req.then );
  });

  test('no query promise resolves with empty array', function () {
    var sd = new SearchData();
    var req = sd.fetch();
    var spy = sinon.spy();

    req.then(spy);

    assert.deepEqual(spy.args[0][0], []);
  });

  test('returns contents of results property of the response', function () {
    var sd = new SearchData();
    var req = sd.fetch('cat');
    var spy = sinon.spy();

    requests[0].respond(
      200, { 'Content-type': 'text/json' },
      JSON.stringify({ results: [ 1, 2, 3 ] })
    );

    req.then(spy);

    assert.deepEqual(spy.args[0][0], [ 1, 2, 3 ]);
  });
});
```

出于简洁性的考虑， 我没有讨论搜索表单的重构， 也简化了些其他的重构和测试。 但是你可以在[这里](https://github.com/rmurphey/testable-javascript)看到完整的版本。

当我们用可测试的js模式重写了程序之后， 一切变得简洁多了：

```
$(function() {
  var pending = false;

  var searchForm = new SearchForm('#searchForm');
  var searchResults = new SearchResults('#results');
  var likes = new Likes('#liked');
  var searchData = new SearchData();

  $(document).on('search', function (event, query) {
    if (pending) { return; }

    pending = true;

    searchData.fetch(query).then(function (results) {
      searchResults.setResults(results);
      pending = false;
    });

    searchResults.pending();
  });

  $(document).on('like', function (evt, name) {
    likes.add(name);
  });
});
```

比获得简洁的代码更重要的是， 现在我们完整的测试了代码。  这样我们就可以放心大胆地重构代码而不用怕会破坏之前写好的成果。 当出现新情况时， 我们甚至可以编写新的
测试用例， 然后编写代码来通过它。


##从长期来说 测试让生活更美好

可能你读到这里，然后突然出现个问题： “等一等！ 你想让我为了同样的功能写更多的代码？”

事实是， 当你在互联网上进行创造时， 有些东西是躲不开的。你会花时间去设计解决问题的方法。你会测试你的解决方案：  或者是通过在浏览器里点击查看反应， 或者是通过自动化测试， 或者是--可怕地--让用户在生产环境中替你做测试。你会修改你的代码， 而其他人会使用你的代码。 而最终： 你的代码会有bug， 无论写了多少测试。

关于测试的一个事实是尽管在开始时会多花费点时间，长远来看它却可以节省时间。当你的测试用例发现了一个bug时，当你有一个运行的系统证明你的bug修改确实有用时，你会觉得庆幸。

##参考资料

这篇文章讨论了Javascript测试的一些基本知识。但是如果你想了解更多，可以参考：

* [我的报告](http://lanyrd.com/2012/full-frontal/sztqh/), 来自2012年伦敦Brighton的Full Frontal会议。
* [Grunt](http://gruntjs.com/)用来进行自动化测试和许多其他任务的工具。
* [编写可测试的Javascript代码](http://www.amazon.com/Test-Driven-JavaScript-Development-Developers-Library/dp/0321683919/ref=sr_1_1?ie=UTF8&qid=1366506174&sr=8-1&keywords=test+driven+javascript+development) 作者Christian Johansen， Sinon库的作者。这是一本关于JS测试的紧凑但信息丰富的书。

##关于作者
###Rebecca Murphey
Rebecca Murphey is a senior software engineer at Bazaarvoice and a frequent speaker on the topics of code organization and best practices at conferences around the world. She’s also the creator of the TXJS conference, the author of the learning site jQuery Fundamentals, a contributor to the jQuery Cookbook from O’Reilly Media, and a technical reviewer for Effective JavaScript by Dave Herman. She blogs at [rmurphey.com](http://lanyrd.com/2012/full-frontal/sztqh/), tweets as [@rmurphey](https://twitter.com/rmurphey), and lives in Durham, NC, with her partner.
