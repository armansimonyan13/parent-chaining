# Parent-Chaining Pattern

## Abstract

[Method chaining](https://martinfowler.com/dslCatalog/methodChaining.html) pattern has been around 
for years in Java and other language. It is widely used to implement [builder](https://martinfowler.com/dslCatalog/constructionBuilder.html) 
or [fluent interface](https://martinfowler.com/bliki/FluentInterface.html)
but not, as far as I know, to instantiate or modify a complete tree structure.

The issue is that the chain termination determines the instance on which the next modifier method will apply on.
For flat structures it is ok but for trees, navigation methods are necessary to go deeper or higher in the structure. 

The purpose of this article is to introduce the *Parent-Chaining* pattern. This pattern completes *method chaining* to 
achieve tree instantiation/modification with *tree-looking-like* code. 

The examples shown here are implemented in Java, but it can apply to any statically typed language featuring generics. 
This repository hosts [source code](src/com/github/djeang/parentchaining) of below examples.

## Example

For developers, the most familiar tree structure is probably the HTML DOM. So let's see how to create 
a HTML DOM instance with an API based on *Parent-Chaining* pattern.

```Java
Html html = new Html()
    .head()
        .title("Title of my document")
        .meta().charset("UTF-8").__
        .meta().content("description", "Parent-Chaining illustrator").__
        .meta().content("keywords", "design pattern, chaining, tree structure, Java").__
        .meta().content("author", "Jerome Angibaud").__.__
    .body()
        .text("This is a simple page for describing Parent-Chaining pattern.")
        .div().attr("style", "bold")
            .text("A first div displayed in bold.")
            .div()
                .img().attr("src", "fuse.png").attr("alt", "A fuse image.").__
                .text("This is a new table")
                .table().attr("style", "width:100%")
                    .tr()
                        .th().text("Firstname").__
                        .th().text("Lastname").__
                        .th().text("Age").__.__
                    .tr()
                        .td().text("Jill").__
                        .td().text("Smith").__
                        .td().text("50").__.__
                    .tr()
                        .td().text("Eve").__
                        .td().text("Jackson").__
                        .td().text("94").__.__.__.__.__
        .div()
            .text("This is the end of this page.").__.__;
```
As you see, the point of this pattern is to write quite readable code when dealing with tree structures 
(which is very common in computer science). Here we were able to fulfill an entire HTML document in a single chained statement. 

You probably noticed the `.__` termination that returns the parent of the current object. 
This is the clue of *Parent-Chaining* pattern.

## Implementation

Let's see now, how to implement it. Let's start by the top one : `Html` class.

```Java
public class Html {

    private final Head head = new Head(this);

    private final TagNode<Html> body = TagNode.ofParent(this, "body");

    public Head head() {
        return head;
    }
    
    public TagNode<Html> body() {
        return body;
    }

}
```
Top element does not need mechanism to return parent, it instantiates its own children by passing them 
itself as reference.

For illustration purpose, `Head` and `Body` does not follow strictly the same design. `<head>` has its own type 
enforcing a restricted set of child tags (<title> or <meta>), while `<body>` and its children use a generic `TagNode` type.
 
```Java
public class Head {

    public final Html __;  // For parent chaining

    private String title;

    private final List<Meta> metas = new LinkedList<>();

    Head(Html parent) {
        this.__ = parent;
    }
```
`Head` only has `Html` as parent, so we don't need of *generics*. In contrast, `TagNode` can have both `TagName`or 
`Html` as parent : we need generics to handle properly this case.

```Java
public class TagNode<P> implements Node {  // P is the genreric type of the parent

    public final P __; // Parent for chaining

    private final String name;  // The tag name : body, div, img, ...

    ...

    protected TagNode(P parent, String name) {
        this.__ = parent;
        this.name = name;
    }

    // To be used in parent-chaining context
    public static <P> TagNode<P> ofParent(P parent, String name) {
        return new TagNode<>(parent, name);
    }

    // To be used outside parent-chaining context
    public static TagNode of(String name) {
        return new TagNode(null, name);
    }
    ...

``` 
The `<P>` generic parameter stands for the type of the parent. Thanks to `of` factory methods, this class can be used 
inside or outside of a parent-chaining pattern.

To append children and attributes to nodes, method `TagNode#child(tagName)` creates a child instance, adds it to its children then return 
the child. To make code more readable, short-cuts has `div()`, `table()`, `tr()`, ... has been implemented.

Note that `child` method returns a `TagNode<TagNode<P>>` and not a `TagNote<T>` as it returns the generic type of the child,
not of itself.

```Java
public TagNode<TagNode<P>> child(String name) {
        TagNode<TagNode<P>> child = TagNode.ofParent(this, name);
        this.children.add(child);
        return child;
    }

    public TagNode<P> text(String text) {
        this.children.add(new TextNode(text));
        return this;
    }

    public TagNode<TagNode<P>> div() {
        return child("div");
    }

    public TagNode<TagNode<P>> img() {
        return child("img");
    }

   ...
```

You can use Java functional consumer to delegate par of the tree handling by a method. `TagLib` implements a 
`apply(Consumer<TagNode)` method for delegation.

```Java
public TagNode<P> apply(Consumer<TagNode<?>> consumer) {
    consumer.accept(this);
    return this;
}
```

So client can handle entire part of the tree in dedicated methods or implement a visitor like pattern.


``` Java
public class MainVariant {

    public static void main(String[] args) {
        Html html = new Html()
            .head()
                .title("Title of my document")
                .meta().charset("UTF-8").__.__
            .body()
                .table().attr("style", "width:100%")
                    .tr()
                        .th().text("Firstname").__
                        .th().text("Lastname").__
                        .th().text("Age").__.__
                    .apply(addRow("Jill", "Smith", "50"))
                    .apply(addRow("Eve", "Jackson", "94")).__
                .apply(MainVariant::addEnding).__;
        System.out.println(html);
    }

    static void addEnding(TagNode tagNode) {
        tagNode
            .div().attr("style", "bold")
                .text("This is the end of this page.");
    }

    static Consumer<TagNode<?>> addRow(String firstname, String lastname, String age) {
        return tagNode -> tagNode
                .tr()
                    .td().text(firstname).__
                    .td().text(lastname).__
                    .td().text(age);
    }
}
```

## Conclusion

*Parent-Chaining* pattern is a solution to impove greatly code readability at a cost of very few extra coding / complexity.

We can imagine XML handling solution based on this pattern to manipulate DOM in a cleaner way or generate better code 
than Jaxb does.

Also version 0.9 of [Jeka](https://dev.jeka) will rely heavily on this pattern to configure project builds.





> Icons made by <a href="https://www.flaticon.com/authors/eucalyp" title="Eucalyp">Eucalyp</a> from <a href="https://www.flaticon.com/" title="Flaticon"> www.flaticon.com</a>
