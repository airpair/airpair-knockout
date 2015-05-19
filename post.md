## Introduction

[Knockout](http://knockoutjs.com/) is a popular JavaScript framework that offers convenient data binding functionality without the complexity of similar frameworks such as [Angular](https://angularjs.org/). It's a great choice for prototyping small applications or introducing data binding into a legacy codebase. This article captures a number of common Knockout pitfalls I've run into over the past few years deploying Knockout code in production. Hopefully it'll save you a headache or two!

__Note:__ All the code in this article can be found on [GitHub](https://github.com/fivethirty/knockout-article-code). Some modifications have been made to examples that contain HTML files so that you can run them in your browser, but the important parts are all the same. If you notice errors in any of the code, please ping me on twitter [@fivethirtyam](https://twitter.com/fivethirtyam) or make a pull request to the GitHub repo.

## 1 Modules

Unlike other UI frameworks, Knockout does not come with a built in way to modularize your application. It's possible to write an entire Knockout app in a single HTML file:

<!-- code lang=markup linenums=true -->

        <html>
            <head>
                <title>My Bad App</title>
                <script type="text/javascript" src="knockout.js"></script>
            </head>
            <body>
                <span data-bind="text: message"></span>
                <script type="text/javascript">
                    function ViewModel() {
                        this.message = ko.observable("This is bad!");
                    }
                    ko.applyBindings(new ViewModel());
                </script>
            </body>
        </html>

While this runs, it definitely won't scale to applications of any significant complexity. The official Knockout docs suggest using [RequireJS](http://requirejs.org/) for [Asynchronous Module Definition (AMD)](http://en.wikipedia.org/wiki/Asynchronous_module_definition), and I wholeheartedly second this recommendation. Using RequireJS and the [domReady](http://requirejs.org/docs/download.html#domReady) module, the above code refactors into three separate files.

Good.html: 

<!-- code lang=markup linenums=true -->

        <html>
            <head>
                <title>My Good App</title>
                <script type="text/javascript" data-main="main.js" src="require.js"></script>
            </head>
            <body>
                <span data-bind="text: message"></span>
            </body>
        </html>
        
main.js:

<!-- code lang=javascript linenums=true -->

        require(['knockout', 'viewModel', 'domReady!'], function(ko, viewModel) {
            ko.applyBindings(new viewModel());
        });
        
viewModel.js:

<!-- code lang=javascript linenums=true -->

        define(['knockout'], function(ko) {
            return function viewModel() {
                this.message = ko.observable("This is good!");
            }
        });


## 2 `var self = this;`

Keeping track of `this` in Knockout can be tough. Setting `var self = this;` at the top of every view model makes it easy.

For example, if `var self = this;` is not set then `this` must be passed into every computed observable:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            this.cat = ko.observable("cat");
            this.pants = ko.observable("pants");
            this.catpants = ko.computed(function() {
                return this.cat() + this.pants();
            }, this);
        }

Forgetting to pass in `this` is easy to do and can result in hard to track down bugs. If `var self = this;` is set then passing `this` is not necessary:

<!-- code lang=javascript linenums=true -->
        
        function viewModel() {
            var self = this;
            
            self.cat = ko.observable("cat");
            self.pants = ko.observable("pants");
            self.catpants = ko.computed(function() {
                return self.cat() + self.pants();
            });
        }
        
`var self = this;` can also help avoid bugs when one binding context refers to another. Consider the following code that displays a list of names. When a user clicks on an name in the list, that name is displayed in the `<span>` below.

HTML:

<!-- code lang=markup linenums=true -->

        <ul data-bind="foreach: people">
            <li data-bind="click: $parent.selectPerson, text: name"></li>
        </ul>
        
        <!-- ko if: selectedPerson -->
            <span data-bind="text: selectedPerson().name"></span>
        <!-- /ko -->

JavaScript:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var self = this;
            
            self.people = [
                {
                    name: "Mike"
                },
                {
                    name: "Sara"
                },
                ...
            ];
            
            self.selectedPerson = ko.observable();
            self.selectPerson = function() {
                // this = the object in people corresponding to the name clicked on
                self.selectedPerson(this);
            };
        }

When `selectPerson` is called its scope is set to the object in `person` corresponding to the name clicked. However, because `var self = this;` is set the view model object is still accessible within the event callback. This would not be the case otherwise.

In production code I almost always define `var self = this;`. However, for the sake of brevity, I won't use it in some examples in this article.

## 3 Logic in templates

Knockout makes it easy to write complicated logic in HTML templates. Avoid doing this at all costs!  It results in difficult to test and maintain code. Instead of using logic in templates...

<!-- code lang=markup linenums=true -->

        <span data-bind="text: firstName() + ' ' + lastName()"></span>

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            this.firstName = ko.observable("Mike");
            this.lastName = ko.observable("Mellenthin");
        };
        
...push all logic into computed observables:

<!-- code lang=markup linenums=true -->

        <span data-bind="text: fullName"></span>
        
<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var self = this;
            
            self.firstName = ko.observable("Mike");
            self.lastName = ko.observable("Mellenthin");
            self.fullName = ko.computed(function() {
                return self.firstName() + ' ' + self.lastName();
            });
        };
        
## 4 Testing

Speaking of testing, by default Knockout offers no tools to help you. This does not mean you shouldn't write tests!  There are [ample](http://jasmine.github.io/) [JS](http://mochajs.org/) [testing](http://karma-runner.github.io/0.12/index.html) frameworks, and you should be able to pick up and drop any of them into your Knockout application. As with anything, the earlier you start writing tests the better. I usually don't start a new Knockout project without also including a testing framework.

## 5 ko.mapping

[ko.mapping](http://knockoutjs.com/documentation/plugins-mapping.html) is a plugin for Knockout that makes working with data fetched from REST endpoints much more enjoyable. 

Without ko.mapping, a view model that consumes objects from a REST API might look like this:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            /*
            * Gets data about a person from the server:
            * person = {
            *   firstName: "John",
            *   lastName: "Stewart"
            * }
            */
            var person = getPersonFromServer();
                
            this.person = {};
            this.person.firstName = ko.observable(person.firstName);
            this.person.lastName = ko.observable(person.lastName);
        }
        
With ko.mapping the view model is much simpler:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var person = getPersonFromServer();
            this.person = ko.mapping.fromJS(person);
        }

Want to convert part of the view model back to plain JSON so that you can POST it back to the API?  That's a one-liner too:

<!-- code lang=javascript linenums=true -->

        var json = ko.mapping.toJS(this.person);
        
While ko.mapping is to some degree officially part of Knockout (it's in the official documentation), it is a separate project that you'll have to download and include.

It's also worth nothing that some people prefer the functionally analogous [ko.viewModel](http://coderenaissance.github.io/knockout.viewmodel/) plugin to ko.mapping. ko.mapping has always worked fine for me, but it only seemed fair to link to both. Use one or the other, but don't use neither!

## 6 textInput and valueUpdate

This may seem minor, but it's caused me so many headaches that it's worth including. Knockout binds the value of input elements a somewhat strange way. To illustrate, let's say we have an input element and want to update some text on the page in real time as the input's value changes:

<!-- code lang=markup linenums=true -->

        <input type="text" data-bind="value: myValue">
        Value: <span data-bind="text: myValue"></span>
        
<!-- code lang=javascript linenums=true -->
        
        function viewModel() {
            this.myValue = ko.observable();
        }
        
Surprisingly, the above won't work. The `myValue` observable (and thus our text in the span) updates whenever the input loses or gains focus, not as it's value changes. Why? For performance reasons this is how Knockout behaves by default.

To fix this in Knockout 3.2 (current at the time of writing) or newer, use the textInput binding instead of the value binding.

<!-- code lang=markup linenums=true -->

        <input type="text" data-bind="textInput: myValue">
        
This will provide immediate updates to `myValue` with the minor caveat that it won't guarantee that the value attribute of the input is always synced with `myValue`.

If you are using an older version of Knockout or if you are dead set on using the value binding, then you must use the `valueUpdate` flag.

<!-- code lang=markup linenums=true -->

        <input type="text" data-bind="value: myValue, valueUpdate='afterkeydown'">
        
In Knockout 3.2 there are 4 possible values for valueUpdate, each with its own idiosyncrasies:

+ \+ `input` - Updates when the value of the element changes. Doesn't work in IE8-.
+ \+ `keyup` - Updates when a keyup event is fired
+ \+ `keypress` - Updates when a keypress event is fired.
+ \+ `afterkeydown` - Updates as soon as the user starts typing a character. Does not work on all mobile browsers -- notably Safari on IOS7.

Using the `textInput` binding is strongly recommended over using the `valueUpdate` flag.

## 7 Don't repeatedly push into observable arrays

Pushing multiple times into observable arrays can cause significant performance issues with your application:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var arr = [0, 1, ..., 999];
            
            this.numbers = ko.observableArray();
            
            for (var i = 0; i < arr.length; i++) {
                this.numbers.push(arr[i]);
            }
        }
        
Running the for loop above will cause Knockout to redraw the page 1000 times -- one for each push. To avoid this, simply overwrite the old value of our entire array:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var arr = [0, 1, ..., 999];
            this.numbers = ko.observableArray(arr);
        }

To transform data before putting it into our new array, use `ko.utils.arrayMap`.

<!-- code lang=javascript linenums=true -->

        function Number(number) {
            this.number = number;
        }
        
        function viewModel() {
            var arr = [0,1, ..., 999];
            
            // creates an array of Number objects
            this.numbers = ko.observableArray(ko.utils.arrayMap(arr, function(number) {
                return new Number(number)
            }));
            
        }

To append many items to an existing observable array, exploit the fact that `push` accepts a variable number of arguments and use `apply` to push them in all at once.

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var arr = [500, 501, ... 999];
            
            this.numbers = ko.observableArray([0, 1, ..., 499]);
            this.numbers.push.apply(self.numbers, arr);
        }
        
## 8 Observable arrays don't automatically have observable members

Observable arrays track changes to _which_ objects are in the array. They do not track changes to the _state_ of those objects.

<!-- code lang=markup linenums=true -->

        <ul data-bind="foreach: people">
            <li data-bind="text: name"></li>
        </ul>
        <button data-bind="click: makeDerekMike">Make Derek Mike</button>

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var self = this;
            
            self.people = ko.observableArray([
                {
                    name: 'Derek',
                },
                {
                    name: 'Sara'
                },
                ...
            ]);
            
            self.makeDerekMike = function() {
                for (var i = 0; i < self.people().length; i++) {
                    if (self.people()[i].name === 'Derek') {
                        self.people()[i].name = 'Mike';
                    }
                }
            };
        }

Clicking the button in the first example will not trigger a page refresh despite the fact that it _will_ modify an object in the array underlying `people`. This is because the same objects still belong to`people`. To trigger a page refresh, we need to explicitly make the fields of the elements in `people` observable:

<!-- code lang=javascript linenums=true -->

        function viewModel() {
            var self = this;
            
            self.people = ko.observableArray([
                {
                    name: ko.observable('Derek')
                },
                {
                    name: ko.observable('Sara')
                },
                ...
            ]);
            
            self.makeDerekMike = function() {
                for (var i = 0; i < self.people().length; i++) {
                    if (self.people()[i].name() === 'Derek') {
                        self.people()[i].name('Mike');
                    }
                }
            };
        }
        
## 9 Components

Because Knockout makes building interactive widgets so simple, it's easy to write similar code in different parts of your application. Use Knockout components to abstract common UI widgets and promote code reuse.

Using components, we can build a simple reusable list builder as follows:

<!-- code lang=markup linenums=true -->

        <span>This is a list with no initial values</span>
        <div data-bind="component: 'list-builder'"></div>
        
        <span>This is a list with some initial values</span>
        <div data-bind="component: {
            name: 'list-builder',
            params: { list: ['item', 'another item'] }
        }"></div>

<!-- code lang=javascript linenums=true -->

        function Item(text) {
            this.text = text;
        }
        
        ko.components.register('list-builder', {
            viewModel: function(params) {
                var self = this;
                
                self.list = ko.observableArray([]);
                
                if (params && params.list) {
                    self.list(ko.utils.arrayMap(params.list, function(item) {
                        return new Item(text);
                    }));
                }
                
                self.newText = ko.observable();
                self.add = function() {
                    self.list.push(new Item(text));
                    self.newText('');
                };
            },
            template:
                '<ul data-bind="foreach: list">' +
                    '<li data-bind="text: text"></li>' +
                '</ul>' +
                '<input type="text" data-bind="textInput: newText" />' +
                '<button data-bind="click: add">Add</button>'
        });
        

Note that you probably don't want to inline the viewModel and template as above in any production code code. Fortunately, this can be solved using require.js (see #1). For a description of how to do this check out the [official components documentation](http://knockoutjs.com/documentation/component-binding.html).

## 10 Tutorials

This sounds like a cop out given that I just wrote a Knockout guide of sorts, but if you're new to Knockout the [official tutorials](http://learn.knockoutjs.com/) are great!  They offer a interactive sandbox to play around with the framework in a guided fashion. They helped me a ton when I was first starting out and hopefully they'll help you too. And if you're still confused, well, there's always AirPair.