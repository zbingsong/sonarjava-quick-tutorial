更新于2023-8-2

# SonarQube自定义规则简便文档

Sonar通过遍历语法树的形式来进行代码检查。

官方的sonar-java插件里已经内置了很多规则，可以在[这里](https://github.com/SonarSource/sonar-java/tree/master/java-checks/src/main/java/org/sonar/java/checks)看到源码。




## 语法树组件

语法树组件包含[树](#树tree)、[类型](#类型type)、[符号](#符号symbol)和[语法token](#语法tokensyntaxtoken)。

### 树`Tree`

整个文件的语法树中的一个节点，表示一个语句、一个表达式、一个变量、一个方法、一个类等等。所有的树类型的组件都可以使用`Tree`类的方法。

`Tree`下有`Tree.Kind`枚举类，包含了所有树节点的种类。注：后面会遇到[类型`Type`](#类型type)，和`Tree.Kind`不是同一个东西。为做区分，将`Tree.Kind`称为种类，将[`Type`](#类型type)称为类型。

`Tree`可以大致分为[表达式树`ExpressionTree`](#表达式树expressiontree)、[语句树`StatementTree`](#语句树statementtree)、[类型树`TypeTree`](#类型树typetree)、[列表树`ListTree`](#列表树listtree)、[模组命令树`ModuleDirectiveTree`](#模组命令树moduledirectivetree)（**Java 9**）和其它树。

* `boolean is(Kind... var1)`：判断这个节点的种类，比如`Tree.Kind.METHOD`、`Tree.Kind.CLASS`等；一次可传入多个类型，如果有一个种类匹配则返回`true`。

* `@Nullable Tree parent()`：获取父节点。

* `@Nullable SyntaxToken firstToken()`：获取这个节点的第一个[`SyntaxToken`](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken lastToken()`：获取这个节点的最后一个[`SyntaxToken`](#语法tokensyntaxtoken)。

* `Tree.Kind kind()`：获取这个节点的种类。


#### 表达式树`ExpressionTree`

父类：[`Tree`](#树tree)

`ExpressionTree`有一系列不同的实现类，比如`LiteralTree`、`MemberSelectExpressionTree`、`NewClassTree`、`MethodInvocationTree`等等。

* `Type symbolType()`：表达式的值的[类型](#类型type)。

* `Optional<Object> asConstant()`：获取表达式的值。

* `<T> Optional<T> asConstant(Class<T> var1)`：获取表达式的值，以`var1`的类型返回。


#### 注解树`AnnotationTree`

父类：[`ExpressionTree`](#表达式树expressiontree)、[`ModifierTree`](#修饰符树modifiertree)

之后会遇到功能相似的[`SymbolMetadata`](#符号元数据symbolmetadata)。`AnnotationTree`和[`SymbolMetadata`](#符号元数据symbolmetadata)的区别在于，`AnnotationTree`的本质是树，可以以树的形式进行自定义的遍历（`AnnotationTree`的父节点是[`ModifierTree`](#修饰符树modifiertree)），而[`SymbolMetadata`](#符号元数据symbolmetadata)可以比较方便地获得注解参数的key和value。

* `SyntaxToken atToken()`：获取注解`@`字符的[语法token](#语法tokensyntaxtoken)。

* `TypeTree annotationType()`：获取注解的[类型树](#类型树typetree)。

* `Arguments arguments()`：获取注解的[参数列表](#参数列表arguments)。


#### 数组元素访问表达式树`ArrayAccessExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

通过下标访问数组的元素的表达式，例如`arr[1]`。

* `ExpressionTree expression()`：获取数组访问的[表达式](#表达式树expressiontree)。

* `ArrayDimensionTree dimension()`：获取数组的[长度](#arraydimensiontree)。


#### 赋值表达式树`AssignmentExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

注意在变量声明时的赋值表达式不算入赋值表达式树中，而算入[变量树](#变量树variabletree)中。例如`int a = 1`中的`a = 1`不算赋值表达式。

* `ExpressionTree variable()`：获取赋值的左边的[表达式](#表达式树expressiontree)。

  _这个[表达式](#表达式树expressiontree)一定只含有变量名。_

* `SyntaxToken operatorToken()`：获取赋值的运算符的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取赋值的右边的[表达式](#表达式树expressiontree)。


#### 一元表达式树`UnaryExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

* `SyntaxToken operatorToken()`：获取运算符的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取运算符后的[表达式](#表达式树expressiontree)。比如对于`int a = -1`，这个方法取到的是`1`。


#### 二元表达式树`BinaryExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

* `ExpressionTree leftOperand()`：获取左操作数的[表达式](#表达式树expressiontree)。

* `SyntaxToken operatorToken()`：获取运算符的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree rightOperand()`：获取右操作数的[表达式](#表达式树expressiontree)。


#### 条件表达式树`ConditionalExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

代表三元运算符`?:`的表达式，即`a ? b : c`。

* `ExpressionTree condition()`：获取条件的[表达式](#表达式树expressiontree)。

* `SyntaxToken questionToken()`：获取问号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree trueExpression()`：获取条件为真时的[表达式](#表达式树expressiontree)。

* `SyntaxToken colonToken()`：获取冒号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree falseExpression()`：获取条件为假时的[表达式](#表达式树expressiontree)。


#### Instanceof树`InstanceOfTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

* `ExpressionTree expression()`：获取被判断的部分的[表达式](#表达式树expressiontree)（即`instanceof`之前的部分）。

* `SyntaxToken instanceofToken()`：获取`instanceof`关键字的[语法token](#语法tokensyntaxtoken)。

* `TypeTree type()`：获取判断的[类型树](#类型树typetree)（即`instanceof`之后的部分）。


#### Lambda表达式树`LambdaExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `@Nullable SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `List<VariableTree> parameters()`：获取lambda表达式的参数列表，以一列[变量树](#变量树variabletree)的方式列出。

* `SyntaxToken arrowToken()`：获取箭头的[语法token](#语法tokensyntaxtoken)。

* `Tree body()`：获取lambda表达式的主体。


##### 字面量树`LiteralTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `String value()`：获取常量的值。

* `SyntaxToken token()`：获取常量的[语法token](#语法tokensyntaxtoken)。


#### 成员访问树`MemberSelectExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 、[`TypeTree`](#类型树typetree)

通过`.`访问成员的表达式，例如`System.out.println()`、`Arrays.asList()`、`classInstance.field`。

注意如果有连续的`.`，那么每个`.`都会有一个`MemberSelectExpressionTree`。例如`java.lang.annotation.*`会被解析为三个`MemberSelectExpressionTree`，分别是`java.lang.annotation.*`、`java.lang.annotation`和`java.lang`。

* `ExpressionTree expression()`：获取访问成员的`.`之前的部分的[表达式](#表达式树expressiontree)。

  例如`System.out.println()`中，这个方法取到的是`System.out`。

* `SyntaxToken operatorToken()`：获取`.`的[语法token](#语法tokensyntaxtoken)。仅限成员名称前面的那个点。

* `IdentifierTree identifier()`：获取被访问的成员的[标识符树](#标识符树identifiertree)。

  例如`System.out.println()`中，这个方法取到的是`println`。


#### 调用方法树`MethodInvocationTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `@Nullable TypeArguments typeArguments()`：获取方法的[泛型参数值列表](#泛型参数值列表typearguments)。

* `ExpressionTree methodSelect()`：获取调用方法的[表达式](#表达式树expressiontree)。

* `Arguments arguments()`：获取调用方法时输入的[参数列表](#参数列表arguments)。

* `Symbol symbol()`：获取被调用的方法的[符号](#符号symbol)。


#### 方法引用树`MethodReferenceTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `Tree expression()`：获取方法引用的表达式。

* `SyntaxToken doubleColon()`：获取`::`的[语法token](#语法tokensyntaxtoken)。

* `@Nullable TypeArguments typeArguments()`：获取方法的[泛型参数值列表](#泛型参数值列表typearguments)。

* `IdentifierTree method()`：获取被引用的方法的[标识符树](#标识符树identifiertree)。


#### 新建数组树`NewArrayTree`

表示创建一个新的数组。

父类：[`ExpressionTree`](#表达式树expressiontree)

* `@Nullable TypeTree type()`：获取数组的[类型树](#类型树typetree)。

* `@Nullable SyntaxToken newKeyword()`：获取`new`关键字的[语法token](#语法tokensyntaxtoken)。

* `List<ArrayDimensionTree> dimensions()`：获取数组的[长度](#数组长度树arraydimensiontree)。

* `@Nullable SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `ListTree<ExpressionTree> initializers()`：获取数组的初始值，以[列表树](#列表树listtree)的方式列出。

* `@Nullable SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### 新建类对象树`NewClassTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

表示创建一个新的类对象。

* `@Nullable ExpressionTree enclosingExpression()`：如果创建对象是在某个语句里进行的（例如`a.method(new MyClass())`），获取外部的[表达式](#表达式树expressiontree)。

* `@Nullable SyntaxToken dotToken()`：获取`.`的[语法token](#语法tokensyntaxtoken)。仅在这个新建对象的类是内部类时才会有。

* `@Nullable SyntaxToken newKeyword()`：获取`new`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable TypeArguments typeArguments()`：获取新建对象时使用的[泛型参数值列表](#泛型参数值列表typearguments)。

* `TypeTree identifier()`：获取类的[类型树](#类型树typetree)。

  `newClassTree.symbolType()`（继承自[`ExpressionTree`](#表达式树expressiontree)的方法）和`newClassTree.identifier().symbolType()`是等价的。

* `Arguments arguments()`：获取新建对象时使用的[参数列表](#参数列表arguments)。

* `@Nullable ClassTree classBody()`：如果新建的对象是一个匿名类，获取类的内容（[类树](#类树classtree)）。

* `Symbol constructorSymbol()`：获取类的`constructor`的[符号](#符号symbol)。


#### 括号树`ParenthesizedTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

一个被括号包起来的表达式，例如`(1 + 2) * 3`中的`(1 + 2)`。

注意仅限于表达式，方法调用、新建类对象、注解等传参使用的括号不包含在这个树中。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取括号中的[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。


#### 类型转换树`TypeCastTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `TypeTree type()`：获取转换的[类型树](#类型树typetree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取被转换的[表达式](#表达式树expressiontree)。

* `@Nullable SyntaxToken andToken()`：获取`&`的[语法token](#语法tokensyntaxtoken)。

  `&`用于[交集类型](#https://docs.oracle.com/javase/8/docs/api/javax/lang/model/type/IntersectionType.html)。交集类型早在**Java 8**中就已经出现，但仅能用于泛型的边界（例如`List<? extends A & B>`）。在**Java 10**中，交集类型的适用范围才被扩展到了普通类型（定义变量、方法返回类型、类型转换等等）。

* `ListTree<Tree> bounds()`：如果转换到的类型中含有泛型，获取这些泛型的边界，用[列表树](#列表树listtree)列出。


#### Switch表达式树`SwitchExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree)、[`SwitchTree`](#switch树switchtree)

无别的方法。


#### 语句树`StatementTree`

父类：[`Tree`](#树tree)

`StatementTree`本身并没有用，也没有`Tree`之外的方法。

它有一系列不同的子类，比如`AssertStatementTree`、`ReturnStatementTree`、`IfStatementTree`，`ForEachStatementTree`等等。


#### Assert语句树`AssertStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken assertKeyword()`：获取assert关键字的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree condition()`：获取assert语句的条件[表达式](#表达式树expressiontree)。

* `@Nullable SyntaxToken colonToken()`：获取冒号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable ExpressionTree detail()`：获取assert语句的详细信息，即冒号后面的表达式。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### Break语句树`BreakStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken breakKeyword()`：获取`break`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable IdentifierTree label()`：获取`break`语句的标签。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 类树`ClassTree`

父类：[`StatementTree`](#语句树statementtree) 

代表定义一个新类。

注意`ClassTree`和[`NewClassTree`](#新建类对象树newclasstree)的区别。`ClassTree`是定义一个新类，而[`NewClassTree`](#新建类对象树newclasstree)是创建一个类的实例。

* `@Nullable SyntaxToken declarationKeyword()`：获取类的声明关键字的[语法token](#语法tokensyntaxtoken)，例如`public`、`private`、`protected`等等。

* `@Nullable SyntaxToken simpleName()`：获取类名的[语法token](#语法tokensyntaxtoken)。

* `TypeParameters typeParameters()`：获取类的[泛型参数列表](#泛型参数列表typeparameters)。

* `ModifiersTree modifiers()`：获取类的修饰符列表，用[多修饰符树](#多修饰符树modifierstree)的方式列出。

* `@Nullable TypeTree superClass()`：获取父类的[类型树](#类型树typetree)。

* `Symbol.TypeSymbol symbol()`：获取类的[类型符号](#类型符号typesymbol)。

* `ListTree<TypeTree> superInterfaces()`：获取类的接口列表，以[列表树](#列表树listtree)的方式列出，元素类型是[类型树](#类型树typetree)。

* `List<Tree> members()`：获取类的成员列表，以一列[树](#树tree)的方式列出。

* `SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### Continue语句树`ContinueStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `SyntaxToken continueKeyword()`：获取`continue`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable IdentifierTree label()`：获取`continue`语句的标签。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### Do-While语句树`DoWhileStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `SyntaxToken doKeyword()`：获取`do`关键字的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取`do`语句的代码块（即`do`后面的语句）。

  _这里把循环里的所有代码（包括大括号）当作一整个语句来处理，实际上返回[`BlockTree`](#块树blocktree)是更合理的做法。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_

* `SyntaxToken whileKeyword()`：获取`while`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree condition()`：获取`while`语句的条件[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 空语句树`EmptyStatementTree`

父类：[`StatementTree`](#语句树statementtree)、[`ImportClauseTree`](#导入项树importclausetree)

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 表达式语句树`ExpressionStatementTree`

父类：[`StatementTree`](#语句树statementtree)

用于表达只含有单个表达式的语句，例如`a += 1`、`System.out.println()`等等。

* `ExpressionTree expression()`：获取[表达式](#表达式树expressiontree)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### For-Each语句`ForEachStatement`

父类：[`StatementTree`](#语句树statementtree)

* `SyntaxToken forKeyword()`：获取`for`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `VariableTree variable()`：获取`for`语句中循环用变量的[变量树](#变量树variabletree)，即`for (Type variable: expression)`中的`variable`。

* `SyntaxToken colonToken()`：获取冒号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取`for`语句中被循环的表达式的[表达式树](#表达式树expressiontree)，即`for (Type variable: expression)`中的`expression`。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取`for`语句后的代码块。

  _这里把循环里的所有代码（包括大括号）当作一整个语句来处理，实际上返回[`BlockTree`](#块树blocktree)是更合理的做法。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_


#### For语句树`ForStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `SyntaxToken forKeyword()`：获取`for`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ListTree<StatementTree> initializer()`：获取`for`语句中的初始化语句，以[列表树](#列表树listtree)的方式列出，元素类型是[语句树](#语句树statementtree)。

* `SyntaxToken firstSemicolonToken()`：获取第一个分号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable ExpressionTree condition()`：获取`for`语句中的条件[表达式](#表达式树expressiontree)。

* `SyntaxToken secondSemicolonToken()`：获取第二个分号的[语法token](#语法tokensyntaxtoken)。

* `ListTree<StatementTree> update()`：获取`for`语句中的更新语句，以[列表树](#列表树listtree)的方式列出，元素类型是[语句树](#语句树statementtree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取`for`语句后的代码块。

  _这里把循环里的所有代码（包括大括号）当作一整个语句来处理，实际上返回[`BlockTree`](#块树blocktree)是更合理的做法。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_


#### If语句树`IfStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

对于含有`else if`的语句，`else if`会被解析成嵌套在`elseStatement`里面的`IfStatementTree`。

例：

```java
if (state == 1) {
  ...
} else if (state == 2) {
  ...
} else if (state == 3) {
  ...
} else {
  ...
}
```

以上的代码会被解析成：

```java
if (state == 1) {
  ...
} else {
  if (state == 2) {
    ...
  } else {
    if (state == 3) {
      ...
    } else {
      ...
    }
  }
}
```

* `SyntaxToken ifKeyword()`：获取if关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree condition()`：获取if语句的条件[表达式](#表达式树expressiontree)。

* `StatementTree thenStatement()`：获取if语句的代码[语句](#语句树statementtree)（即如果if条件为真时会执行的语句）。

  _虽然返回值是[`StatementTree`](#语句树statementtree)，但其实返回[`BlockTree`](#块树blocktree)更合理；这个返回的[`StatementTree`](#语句树statementtree)把大括号中（包括大括号本身）所有的语句都当作一个语句来处理。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_

* `@Nullable SyntaxToken elseKeyword()`：获取else关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable StatementTree elseStatement()`：获取else语句的代码[语句](#语句树statementtree)（即如果if条件为假时会执行的语句）。

  _虽然返回值是[`StatementTree`](#语句树statementtree)，但其实返回[`BlockTree`](#块树blocktree)更合理；这个返回的[`StatementTree`](#语句树statementtree)把大括号中（包括大括号本身）所有的语句都当作一个语句来处理。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_


#### Labeled语句树`LabeledStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `IdentifierTree label()`：获取标签的[标识符树](#标识符树identifiertree)。

* `SyntaxToken colonToken()`：获取冒号的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取标签后的[语句](#语句树statementtree)。

* `Symbol.LabelSymbol symbol()`：获取标签的[类型符号](#类型符号typesymbol)。


#### Return语句树`ReturnStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `SyntaxToken returnKeyword()`：获取`return`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable ExpressionTree expression()`：获取`return`语句的返回[表达式](#表达式树expressiontree)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


##### Switch语句树`SwitchStatementTree`

父类：[`StatementTree`](#语句树statementtree) 、[`SwitchTree`](#switch树switchtree)

本身没有方法，所用的方法全部来自于[`SwitchTree`](#switch树switchtree)。


#### Synchronized语句树`SynchronizedStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

仅限于`synchronized`代码块，不包括`synchronized`方法。

* `SyntaxToken synchronizedKeyword()`：获取`synchronized`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取`synchronized`语句括号里的[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `BlockTree block()`：获取`synchronized`后的代码[块树](#块树blocktree)。


#### Throw语句树`ThrowStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken throwKeyword()`：获取`throw`关键字的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取`throw`后面语句的[表达式](#表达式树expressiontree)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### Try语句树`TryStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken tryKeyword()`：获取`try`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `ListTree<Tree> resourceList()`：获取`try`语句括号中的资源列表，以[列表树](#列表树listtree)的方式列出。

* `BlockTree block()`：获取`try`后的代码[块树](#块树blocktree)。

* `List<CatchTree> catches()`：获取`try`后的所有`catch`语句，以一列[catch语句树](#catch语句树catchstatementtree)的方式列出。

* `@Nullable SyntaxToken finallyKeyword()`：获取`finally`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable BlockTree finallyBlock()`：获取`try`后的`finally`语句的代码[块树](#块树blocktree)。


#### While语句树`WhileStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken whileKeyword()`：获取`while`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree condition()`：获取`while`语句的条件[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取`while`语句后的代码[语句](#语句树statementtree)。

  _这里把循环里的所有代码（包括大括号）当作一整个语句来处理，实际上返回[`BlockTree`](#块树blocktree)是更合理的做法。我们依然可以通过自定义遍历器的方式访问代码块的内容，详见[自定义规则范例](#for循环和for-each循环里不能嵌套if)。_


#### Yield语句树`YieldStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

`yield`关键字是**Java 13**中引入、**Java 14**中正式确定的，用于在`switch`语句中返回值。详见[Java 17更新文档](https://docs.oracle.com/en/java/javase/17/language/switch-expressions.html#GUID-BA4F63E3-4823-43C6-A5F3-BAA4A2EF3ADC)。

例：

```java
String result = switch (expression) {
    case 1 -> yield "One";
    case 2 -> yield "Two";
    default -> yield "Other";
};
```

如果`switch`语句中所有的`case`都会返回值，那么`yield`关键字就可以被省略：

```java
String result = switch (expression) {
    case 1 -> "One";
    case 2 -> "Two";
    default -> "Other";
};
```

* `@Nullable SyntaxToken yieldKeyword()`：获取`yield`关键字的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取`yield`语句的返回[表达式](#表达式树expressiontree)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 变量树`VariableTree`

父类：[`StatementTree`](#语句树statementtree)

* `ModifiersTree modifiers()`：获取变量的[多修饰符树](#多修饰符树modifierstree)。

* `TypeTree type()`：获取变量的[类型树](#类型树typetree)。

* `IdentifierTree simpleName()`：获取变量名的[标识符树](#标识符树identifiertree)。

* `@Nullable SyntaxToken equalToken()`：如果变量在被声明时就赋值了，获取赋值语句中等号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable ExpressionTree initializer()`：如果变量在被声明时就赋值了，获取赋值语句中初始值的[表达式](#表达式树expressiontree)。

  注意在声明时的赋值表达式不算[赋值表达式树](#赋值表达式树assignmentexpressiontree)。

* `Symbol symbol()`：获取变量的[符号](#符号symbol)。

* `@Nullable SyntaxToken endToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 块树`BlockTree`

父类：[`StatementTree`](#语句树statementtree)

使用`BlockTree`的树和方法很少。

* `List<StatementTree> body()`：获取块的内容，以一列[语句树](#语句树statementtree)的方式列出。

* `SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### 静态初始化块树`StaticInitializerTree`

父类：[`BlockTree`](#块树blocktree)

代表一个类中的静态初始化代码块。

例：

```java

class MyClass {
  private static final int a;
  // 静态初始化代码块
  static {
    a = 1;
  }
}
```

* `SyntaxToken staticKeyword()`：获取`static`关键字的[语法token](#语法tokensyntaxtoken)。


#### 方法树`MethodTree`

父类：[`Tree`](#树tree)

* `ModifiersTree modifiers()`：获取方法的[多修饰符树](#多修饰符树modifierstree)。

* `TypeParameters typeParameters()`：获取这个方法的[泛型参数列表](#泛型参数列表typeparameters)。

* `@Nullable TypeTree returnType()`：获取这个方法的返回值的[类型树](#类型树typetree)。

* `IdentifierTree simpleName()`：获取方法名的[标识符树](#标识符树identifiertree)。

* `List<VariableTree> parameters()`：获取这个方法的参数列表，以一列[变量树](#变量树variabletree)的方式列出。

* `SyntaxToken throwsToken()`：获取这个方法的`throws`关键字的[语法token](#语法tokensyntaxtoken)。

* `ListTree<TypeTree> throwsClauses()`：获取`throws`后面跟着的异常的[类型树](#类型树typetree)，以[列表树](#列表树listtree)的方式列出，元素类型是[类型树](#类型树typetree)。

* `@Nullable BlockTree block()`：以[块树](#块树blocktree)的方式获取这个方法的内容。

* `@Nullable SyntaxToken defaultToken()`：获取这个方法的`default`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken openParenToken()`：获取这个方法的左括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeParenToken()`：获取这个方法的右括号的[语法token](#语法tokensyntaxtoken)。

* `Symbol.MethodSymbol symbol()`：获取这个方法的[方法符号](#方法符号methodsymbol)。


#### 类型树`TypeTree`

父类：[`Tree`](#树tree)

把节点的类型用树的方式存储。相比起[类型](#类型type)，类型树包含了更丰富的信息。

* `Type symbolType()`：获取类型树代表的[类型](#类型type)。

* `List<AnnotationTree>`：获取类型树的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。


#### 数组类型树`ArrayTypeTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 、[`TypeTree`](#类型树typetree)

代表数组类型。注意方法参数中的可变参数也算作是数组类型。

* `TypeTree type()`：获取数组的[类型树](#类型树typetree)。

* `@Nullable SyntaxToken openBracketToken()`：获取左中括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeBracketToken()`：获取右中括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken ellipsisToken()`：如果这个数组类型是可变参数，获取省略号的[语法token](#语法tokensyntaxtoken)。


#### 标识符树`IdentifierTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 、[`TypeTree`](#类型树typetree)

* `SyntaxToken identifierToken()`：获取标识符的[语法token](#语法tokensyntaxtoken)。

* `String name()`：标识符的名称。

* `Symbol symbol()`：标识符的[符号](#符号symbol)。


#### 含参类型树`ParameterizedTypeTree`

父类：[`TypeTree`](#类型树typetree)

用于表示一个泛型。

* `TypeTree type()`：获取[类型树](#类型树typetree)。

* `TypeArguments typeArguments()`：获取类型的[泛型参数值列表](#泛型参数值列表typearguments)。


#### 原始类型树`PrimitiveTypeTree`

父类：[`ExpressionTree`](#表达式树expressiontree)、[`TypeTree`](#类型树typetree)

* `SyntaxToken keyword()`：获取原始类型的[语法token](#语法tokensyntaxtoken)（例如`int`、`byte`等等）。


#### 联合类型树`UnionTypeTree`

父类：[`TypeTree`](#类型树typetree)

`UnionType`是**Java 14**中的新特性，表示一个联合类型，与TypeScript中的联合类型类似。详见[`UnionType`文档](https://docs.oracle.com/en/java/javase/14/docs/api/java.compiler/javax/lang/model/type/UnionType.html)。

与TypeScript的联合类型不同的是，Java中的`UnionType`的使用范围非常局限，只能用于异常的`catch`语句中。通常来说，当需要捕获多个异常时，我们需要写多个`catch`语句，即使对这些异常的处理方式是一样的。而使用`UnionType`可以将多个异常的捕获写在一个`catch`语句中，使得代码看起来更加精简。注意`UnionType`中列出的异常类型必须是互不相交的，即不能有继承关系。

例：

```java
public class MultiCatchExample {
  public static void main(String[] args) {
    try {
      int[] arr = {1, 2, 3};
      System.out.println(arr[5]);
    } catch (ArrayIndexOutOfBoundsException | NullPointerException e) {
      System.out.println("Exception caught: " + e);
    }
  }
}
```

在上面的例子中，我们同时捕获了`ArrayIndexOutOfBoundsException`和`NullPointerException`，并且用同样的方式处理了这两个异常。

* `ListTree<TypeTree> typeAlternatives()`：获取联合类型中的所有类型，以[列表树](#列表树listtree)的方式列出。


#### var类型树`VarTypeTree`

父类：[`TypeTree`](#类型树typetree)

`var`关键字是**Java 10**中的新特性。当声明一个局部变量时，我们需要会指定它的类型。而在Java 10中，我们可以把局部变量的类型声明为`var`，让Java自己根据上下文去猜测这个变量的类型。详见[Java 10更新描述](https://docs.oracle.com/javase/10/language/toc.htm#JSLAN-GUID-7D5FDD65-ACE4-4B3C-80F4-CC01CBD211A4)。

* `SyntaxToken varToken()`：获取`var`关键字的[语法token](#语法tokensyntaxtoken)。


#### 通配符树`WildcardTree`

父类：[`TypeTree`](#类型树typetree)

用于检测泛型中的通配符。

* `List<AnnotationTree> annotations()`：获取泛型的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

* `SyntaxToken queryToken()`：获取通配符`?`的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken extendsOrSuperToken()`：获取`extends`或`super`的[语法token](#语法tokensyntaxtoken)。

* `@Nullable TypeTree bound()`：获取通配符的边界的[类型树](#类型树typetree)。


#### 列表树`ListTree`

父类：[`Tree`](#树tree)、`List`

`ListTree`既能当作[`Tree`](#树tree)使用，也能当作`List`使用。

* `List<SyntaxToken> separators()`：获取列表中的分隔符，分隔符以一列[语法token](#语法tokensyntaxtoken)的方式列出。

  例如`ListTree`中有`a`、`b`、`c`三个元素，那么`separators()`返回的就是`a`和`b`之间的分隔符以及`b`和`c`之间的分隔符。


#### 多修饰符树`ModifiersTree`

父类：[`ListTree`](#列表树listtree)（元素类型为[ModifierTree](#修饰符树`ModifierTree`)）

表示一系列的修饰符。

`ModifiersTree`是[`ListTree`](#列表树listtree)的子类，可以直接当作一个包含[ModifierTree](#修饰符树`ModifierTree`)的`List`来使用。

* `List<AnnotationTree> annotations()`：获取列表中的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

* `List<ModifierKeywordTree> modifiers()`：获取列表中的修饰符，修饰符以一列[修饰符关键字树](#修饰符关键字树modifierkeywordtree)的方式列出。

  用这个方法和直接把`ModifiersTree`当作`List`使用的区别是`List`中元素的类型。这个方法返回的是[`ModifierKeywordTree`](#修饰符关键字树modifierkeywordtree)，而直接当作`List`返回的则是`ModifierTree`。


#### 参数列表`Arguments`

父类：[`ListTree`](#列表树listtree)（元素类型为[`ExpressionTree`](#表达式树expressiontree)）

获取一系列参数的值。

* `@Nullable SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。


#### 泛型参数值列表`TypeArguments`

父类：[`ListTree`](#列表树listtree)（元素类型为[`Tree`](#树tree)）

获取一系列泛型参数的值。

`TypeArguments`与[`TypeParameters`](#泛型参数列表typeparameters)的区别在于，`TypeArguments`中的泛型参数是具体的类型（即泛型参数值），而[`TypeParameters`](#泛型参数列表typeparameters)中的泛型参数是占位符的形式。

例：

```java
public class Box<T extends Number> {
    private T value;

    public Box(T value) {
        this.value = value;
    }
}

public class TypeArgumentsVsTypeParameters {
  public static void main(String[] args) {
      Box<Integer> stringBox = new Box<>(123);
  }
}
```

这个例子中，`Box<String>`中的`String`是泛型参数值，属于`TypeArguments`，而`Box<T extends Number>`中的`T extends Number`是泛型参数，属于[`TypeParameters`](#泛型参数列表typeparameters)。

* `SyntaxToken openBracketToken()`：获取泛型参数列表的左尖括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken closeBracketToken()`：获取泛型参数列表的右尖括号的[语法token](#语法tokensyntaxtoken)。


#### 泛型参数列表`TypeParameters`

父类：[`ListTree`](#列表树listtree)（元素类型为[`TypeParameterTree`](#类型参数树typeparametertree)）

`TypeParameters`与[`TypeArguments`](#泛型参数值列表typearguments)的区别见[`TypeArguments`](#泛型参数值列表typearguments)下的说明。

* `@Nullable SyntaxToken openBracketToken()`：获取泛型参数列表的左尖括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeBracketToken()`：获取泛型参数列表的右尖括号的[语法token](#语法tokensyntaxtoken)。


#### 模组名称树`ModuleNameTree`

父类：[`ListTree`](#列表树listtree)（元素类型为[`IdentifierTree`](#标识符树identifiertree)）

相比父类没有任何新增的方法。

关于模组的更多信息见[`ModuleDirectiveTree`](#模组命令树moduledirectivetree)。


#### 修饰符树`ModifierTree`

父类：[`Tree`](#树tree)

相比它的父类[`Tree`](#树tree)没有任何新加的方法。


#### 修饰符关键字树`ModifierKeywordTree`

父类：[`ModifierTree`](#修饰符树modifiertree)

* `Modifier modifier()`：获取修饰符的类型。

  `Modifier`是一个枚举类，包含了所有修饰符的类型。

* `SyntaxToken keyword()`：获取修饰符的[语法token](#语法tokensyntaxtoken)。


#### 泛型参数树`TypeParameterTree`

父类：[`Tree`](#树tree)

泛型参数是用来表示泛型的占位符，例如`MyClass<T extends Number>`中的`T extends Number`。

* `IdentifierTree identifier()`：获取泛型参数的[标识符树](#标识符树identifiertree)。

* `@Nullable SyntaxToken extendsToken()`：获取`extends`关键字的[语法token](#语法tokensyntaxtoken)。

* `ListTree<Tree> bounds()`：获取泛型参数的边界，边界以[列表树](#列表树listtree)的方式列出。


#### 模组命令树`ModuleDirectiveTree`

父类：[`Tree`](#树tree)

模组`module`是**Java 9**中的新特性，用于模块化Java程序。详见[Java 9模组说明文章](https://www.oracle.com/corporate/features/understanding-java-9-modules.html)和[第三方教程](https://jenkov.com/tutorials/java/modules.html)。

构建模组的文件结构如下：

```
模组名
├─module-info.java
├─package1
├─package2
└─...
```

在`module-info.java`中需要声明这个模组以及它的依赖关系。`module-info.java`的内容如下：

```java
module com.example.mymodule {
  // 模组导出的包，只有这里导出的包才能被别的模组使用
  exports com.example.package1;
  exports com.example.package2;
  exports ...

  // 模组的依赖
  // static表示这个依赖只在编译时需要，运行时不需要
  // transitive表示如果别的模组导入了这个模组（mymodule），这个依赖（dependency1）也会被别的模组作为依赖导入
  requires [static | transitive] com.example.dependency1;
  requires com.example.dependency2.service;
  requires com.example.dependency3.implementedservice;

  // 如果需要实现一个服务模组的接口
  provides com.example.dependency2.service.ServiceInterface with
    com.example.service.ServiceInterfaceImpl
   
  // 如果需要使用一个已经实现的服务模组
  uses com.example.dependency3.implementedservice.ServiceInterface

  // 如果需要用反射获取一个包中的所有成员，包括private成员
  opens com.example.dependency1;
}
```

* `SyntaxToken directiveKeyword()`：获取命令的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### `RequiresDirectiveTree`

用于指定一个依赖的模组。

父类：[`ModuleDirectiveTree`](#moduledirectivetree)

* `ModifiersTree modifiers()`：获取`require`命令的[多修饰符树](#多修饰符树modifierstree)。

* `ModuleNameTree moduleName()`：获取`require`命令后的所有模组，以[模组名称树](#模组名称树modulenametree)的方式列出。


#### `ExportsDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)

用于指定一个包作为模组的导出项。

* `ExpressionTree packageName()`：获取`exports`命令后的包的[表达式树](#表达式树expressiontree)。

* `@Nullable SyntaxToken toKeyword()`：获取`exports`命令后的`to`关键字的[语法token](#语法tokensyntaxtoken)。

  `to`关键字可以指定这个包只能被哪些模组使用。如果没有指定`to`关键字，那么这个包就可以被所有模组使用。

* `ListTree<ModuleNameTree> moduleNames()`：获取`exports`命令中`to`关键字后的所有模组，以[列表树](#列表树listtree)的方式列出，元素类型是[模组名称树](#模组名称树modulenametree)。


#### `OpensDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)

用于指定一个包作为模组的开放项，这个包里的所有成员都可以在这个模组中用反射获取。在**Java 8**和更早版本中，反射总是能获取一个包中的所有成员，包括`private`成员。但是在**Java 9**和更高版本中，反射默认只能获取除`private`以外的成员，如果想要获取`private`成员，就需要使用`opens`命令。

* `ExpressionTree packageName()`：获取`opens`命令后的包的[表达式树](#表达式树expressiontree)。

* `@Nullable SyntaxToken toKeyword()`：获取`opens`命令后的`to`关键字的[语法token](#语法tokensyntaxtoken)。

  `to`关键字可以指定这个包只能被哪些模组使用。如果没有指定`to`关键字，那么这个包就可以被所有模组使用。

* `ListTree<ModuleNameTree> moduleNames()`：获取`opens`命令中`to`关键字后的所有模组，以[列表树](#列表树listtree)的方式列出，元素类型是[模组名称树](#模组名称树modulenametree)。


#### `UsesDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)

用于指定一个可供使用的服务模组接口。

* `TypeTree typeName()`：获取`uses`命令后的服务模组接口的[类型树](#类型树typetree)。


#### `ProvidesDirectiveTree`

用于给一个服务模组接口提供一个实现。

父类：[`ModuleDirectiveTree`](#moduledirectivetree)

* `TypeTree typeName()`：获取`provides`命令后的服务模组接口的[类型树](#类型树typetree)。

* `SyntaxToken withKeyword()`：获取`provides`命令后的`with`关键字的[语法token](#语法tokensyntaxtoken)。

* `ListTree<TypeTree> typeNames()`：获取`provides`命令中`with`关键字后的所有服务模组接口的实现类，以[列表树](#列表树listtree)的方式列出，元素类型是[类型树](#类型树typetree)。


#### 编译单元树`CompilationUnitTree`

父类：[`Tree`](#树tree)

编译单元是指一个Java源文件，例如`MyClass.java`，编译单元树是指这个源文件的语法树。编译单元树一般是所有树的最终父节点。

* `@Nullable PackageDeclarationTree packageDeclaration()`：获取编译单元中的包声明，以[包声明树](#包声明树packagedeclarationtree)的方式获取。

* `List<ImportClauseTree> imports()`：获取编译单元中的所有导入语句，以一列[导入项树](#导入项树importclausetree)的方式列出。

* `List<Tree> types()`：获取编译单元中的所有类型声明，以一列[树](#树tree)的方式列出。

* `@Nullable ModuleDeclarationTree moduleDeclaration()`：获取编译单元中的模组声明，以[模组声明树](#模组声明树moduledeclarationtree)的方式获取。

* `SyntaxToken eofToken()`：获取编译单元的结束符的[语法token](#语法tokensyntaxtoken)。


#### 导入项树`ImportClauseTree`

父类：[`Tree`](#树tree)

与父类相比没有新增的方法。

有时能类型转换成[`ImportTree`](#导入树importtree)。


#### 导入树`ImportTree`

父类：[`ImportClauseTree`](#导入项树importclausetree)

* `boolean isStatic()`：判断这个导入项是否是静态导入。

* `SyntaxToken importKeyword()`：获取`import`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken staticKeyword()`：如果这个导入项是静态导入，获取`static`关键字的[语法token](#语法tokensyntaxtoken)。

* `Tree qualifiedIdentifier()`：获取导入的包。

  _这个地方的[`Tree`](#树tree)可以转换为[`MemberSelectExpressionTree`](#成员访问树memberselectexpressiontree)或者[`IdentifierTree`](#标识符树identifiertree)。_

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### 模组声明树`ModuleDeclarationTree`

父类：[`Tree`](#树tree)

关于模组的更多信息见[`ModuleDirectiveTree`](#模组命令树moduledirectivetree)。

* `List<AnnotationTree> annotations()`：获取模组声明的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

  _注意：模组声明的注解只能是`@Deprecated`。_

* `@Nullable SyntaxToken openKeyword()`：获取`open`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken moduleKeyword()`：获取`module`关键字的[语法token](#语法tokensyntaxtoken)。

* `ModuleNameTree moduleName()`：获取模组名的[模组名称树](#模组名称树modulenametree)。

* `SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `List<ModuleDirectiveTree> directives()`：获取模组中的所有命令，以一列[模组命令树](#模组命令树moduledirectivetree)的方式列出。

* `SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### 包声明树`PackageDeclarationTree`

父类：[`Tree`](#树tree)

* `List<AnnotationTree> annotations()`：获取包声明的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

  _注意：包声明的注解只能是`@Deprecated`。_

* `SyntaxToken packageKeyword()`：获取`package`关键字的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree packageName()`：获取包名的[表达式树](#表达式树expressiontree)。

* `SyntaxToken semicolonToken()`：获取分号的[语法token](#语法tokensyntaxtoken)。


#### Switch树`SwitchTree`

父类：[`Tree`](#树tree)

* `SyntaxToken switchKeyword()`：获取`switch`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取`switch`语句的条件[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `List<CaseGroupTree> cases()`：获取`switch`语句的`case`组，以一列[Case组树](#case组树casegrouptree)的方式列出。

* `SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### Case组树`CaseGroupTree`

父类：[`Tree`](#树tree)

* `List<CaseLabelTree> labels()`：获取`case`组的标签，以一列[Case标签树](#case标签树caselabeltree)的方式列出。

* `List<StatementTree> body()`：获取`case`组的内容，以一列[语句树](#语句树statementtree)的方式列出。


#### Case标签树`CaseLabelTree`

父类：[`Tree`](#树tree)

* `SyntaxToken caseOrDefaultKeyword()`：获取`case`或`default`关键字的[语法token](#语法tokensyntaxtoken)。

* `boolean isFallThrough()`：判断这一个`case`是否会穿透到下一个`case`。

* `List<ExpressionTree> expression()`：获取`case`标签的所有条件[表达式](#表达式树expressiontree)。

* `SyntaxToken colonOrArrowToken()`：获取冒号或箭头的[语法token](#语法tokensyntaxtoken)。

  一般来说，`switch`中的`case`语句的条件后会跟一个冒号。在**Java 12**或更高版本中，这个冒号可以使用箭头`->`来代替。使用箭头的`case`在执行完成后会自动`break`，避免因为忘记写`break`而导致的穿透问题。详见[Java 12博客](https://blogs.oracle.com/javamagazine/post/new-switch-expressions-in-java-12)和[Java 17更新文档](https://docs.oracle.com/en/java/javase/17/language/switch-expressions.html#GUID-BA4F63E3-4823-43C6-A5F3-BAA4A2EF3ADC)。


#### Catch树`CatchTree`

父类：[`Tree`](#树tree)

* `SyntaxToken catchKeyword()`：获取`catch`关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `VariableTree parameter()`：获取`catch`语句的参数，以[变量树](#变量树variableTree)的方式获取。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `BlockTree block()`：获取`catch`语句的内容，以[块树](#块树blocktree)的方式获取。


#### Enum常量树`EnumConstantTree`

父类：[`Tree`](#树tree)

代表一个`enum`中的一个常量。

* `ModifiersTree modifiers()`：获取`enum`常量的[多修饰符树](#多修饰符树modifierstree)。

  _不知道有什么用，一个`enum`常量是没有修饰符的。_

* `IdentifierTree simpleName()`：获取`enum`常量的[标识符树](#标识符树identifiertree)。

* `NewClassTree initializer()`：获取`enum`常量的初始化语句，以[新建类树](#%E6%96%B0%E5%BB%BA%E7%B1%BB%E5%AF%B9%E8%B1%A1%E6%A0%91newclasstree)的方式获取。

* `@Nullable SyntaxToken separatorToken()`：获取`enum`常量后的分隔符的[语法token](#语法tokensyntaxtoken)。

  `enum`常量之间的分隔符是逗号，最后一个常量后的分隔符是分号。


#### 数组长度树`ArrayDimensionTree`

父类：[`Tree`](#树tree)

代表一个数组在一个维度上的长度。例如`new int[5][10]`中的`[5]`和`[10]`分别是一个数组长度树。

* `List<AnnotationTree> annotations()`：获取数组长度的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

* `SyntaxToken openBracketToken()`：获取左方括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable ExpressionTree expression()`：获取数组长度的[表达式树](#表达式树expressiontree)。

* `SyntaxToken closeBracketToken()`：获取右方括号的[语法token](#语法tokensyntaxtoken)。



### 类型`Type`

父类：无

类型是把Java中的类型转换为语法树元素得来的，内置了多种判断类型的方法。

注意

1. 此处的类型`Type`和[`Tree.Kind`](#树tree)中的种类不是同一个东西：此处的类型是Java中的类型，[`Tree.Kind`](#树tree)中的种类是语法树中的种类。

2. `import`语句中的类型是无法获取的。例如`import java.util.List`，`List`是无法获取的。

`Type`下有`Type.Primitives`枚举类，包含了所有Java的基本类型。

* `boolean is(String var1)`：判断类型是否是`var1`，`var1`是另一个类型的名称的字符串

  对比类型时要用这个方法，而不能直接对类型用`==`或`equals`。

  例：`myType.is("java.lang.String")`。

* `boolean is...()`：对类型进行判断，比如`isClass()`、`isPrimitive()`、`isArray()`等等。

* `boolean isSubtypeOf(String var1)`：判断类型是否是`var1`的子类型，`var1`是另一个类型的名称的字符串。

* `boolean isSubtypeOf(Type var1)`：判断类型是否是`var1`的子类型，`var1`是另一个类型。

* `String fullyQualifiedName()`：获取类型的全名，例如`java.lang.String`。

* `String name()`：获取类型的名称。

* `Symbol.TypeSymbol symbol()`：获取类型的[类型符号](#类型符号typesymbol)。

* `Type erasure()`：获取类型的擦除类型。

* `boolean isParameterized()`：判断类型是否是参数化类型（即泛型）。

* `List<Type> typeArguments()`：获取类型的参数类型列表。


#### 数组类型`ArrayType`

父类：[`Type`](#类型Type)

* `Type elementType()`：获取数组元素的类型。



### 符号`Symbol`

父类：无

符号表示一个方法、类、变量等等的签名信息。符号是SonarQube中的符号化（symbolic）API的核心，它可以“假装运行”代码来理清代码逻辑和流向控制，追踪变量的使用和方法的调用等等。

`Symbol`有一个类似的类[`LabelSymbol`](#标签符号labelsymbol)，三个子类[`MethodSymbol`](#方法符号methodsymbol)、[`VariableSymbol`](#变量符号variablesymbol)、[`TypeSymbol`](#类型符号typesymbol)，和一个相关的类[`SymbolMetadata`](#符号元数据symbolmetadata)。

* `String name()`：获取符号的名称。

* `Type type()`：获取符号的[类型](#类型type)。

* `boolean is...()`：判断符号的类型，比如`isMethodSymbol()`、`isVariableSymbol()`、`isTypeSymbol()`等等；也可以判断符号的属性，比如`isPublic()`、`isPrivate()`、`isStatic()`等等。

* `SymbolMetadata metadata()`：获取符号的[符号元数据](#符号元数据symbolmetadata)，可以获取符号的注解等信息。

* `List<IdentifierTree> usages()`：获取符号的使用列表，为一个[标识符](#标识符树identifiertree)列表。

* `Tree declaration()`：符号的声明，在各子类中有对应重载的方法。


#### 方法符号`MethodSymbol`

父类：[`Symbol`](#符号Symbol)

* `List<Type> parameterTypes()`：获取方法的参数[类型](#类型type)列表。

* `TypeSymbol returnType()`：获取方法的返回值[类型符号](#类型符号TypeSymbol)。

* `List<Type> thrownTypes()`：获取方法抛出的异常中的[类型](#类型type)列表。

* `List<MethodSymbol> overriddenSymbols()`：获取方法的重写[方法符号](#方法符号methodsymbol)列表。

* `String signature()`：获取方法的签名。

* `@Nullable MethodTree declaration()`：以[方法树](#方法树methodtree)的形式，获取方法的声明。


#### 变量符号`VariableSymbol`

父类：[`Symbol`](#符号Symbol)

* `@Nullable VariableTree declaration()`：以[变量树](#变量树variableTree)的形式，获取变量的声明。


#### 类型符号`TypeSymbol`

父类：[`Symbol`](#符号Symbol)

用于类（`class`）的符号。

* `@CheckForNull Type superClass()`：获取类型的父[类型](#类型type)。

* `List<Type> interfaces()`：获取类型实现的接口[类型](#类型type)列表。

* `Collection<Symbol> memberSymbols()`：获取类型的成员[符号](#符号symbol)列表。

* `Collection<Symbol> lookupSymbols(String var1);`：在类型的成员中搜索名称为`var1`的[符号](#符号symbol)并返回匹配。

* `@Nullable ClassTree declaration()`：以[类树](#类树classtree)的形式，获取类型的声明。


#### 标签符号`LabelSymbol`

父类：无

注意`LabelSymbol`不是`Symbol`的子类，而是`Symbol`的另一个类似类，所以它无法使用`Symbol`的方法。

* `String name()`：获取标签的名称。

* `List<IdentifierTree> usages()`：获取标签的使用，为一个[标识符](#标识符树identifiertree)列表。

* `@Nullable LabeledStatementTree declaration()`：获取标签的声明。


#### 符号元数据`SymbolMetadata`

父类：无

表示符号的元数据，用于获取符号的注解信息。注意[`AnnotationTree`](#注解树annotationtree)同样可以获取注解信息，这两者的区别见[`AnnotationTree`](#注解树annotationtree)下的说明。

`SymbolMetadata`中定义了两个接口，`AnnotationInstance`和`AnnotationValue`，分别用于获取单个注解和单个注解的值。

* `boolean isAnnotatedWith(String annotationName)`：判断符号是否被`annotationName`注解了。

* `@CheckForNull List<AnnotationValue> valuesForAnnotation(String annotationName)`：获取用于注解这个符号的，名称是`annotationName`的注解的参数。

* `List<AnnotationInstance> annotations()`：获取符号的所有[注解实例](#注解实例annotationinstance)。

##### 注解实例`AnnotationInstance`

父类：无

表示一个注解。

* `Symbol symbol()`：获取注解的[符号](#符号symbol)。

* `List<AnnotationValue> values()`：获取[注解值](#注解值annotationvalue)列表。

##### 注解值`AnnotationValue`

父类：无

表示一个注解值（包括key和value）。

* `String name()`：获取注解值的键值（key）。

* `Object value()`：获取注解值的值（value）。


### 语法token`SyntaxToken`

父类：[`Tree`](#树Tree)

一个`SyntaxToken`表示一个词语或者符号，比如`return`、`String`、`==`、`+`、`{`、`}`等等。使用token可以有效检查代码格式，比如判断是否有空格、是否有换行等等。token也是SonarJava中获取代码源文本以及文本位置的唯一方式。

* `String text()`：token的原文本。

* `int line()`：token所在的行数。

* `int column()`：token所在的列数。

* `List<SyntaxTrivia>`：token的注释。

  _暂时没发现怎么使用，测试中无法获取到注释。_




## 自定义规则

以下的文件中，[规则Java文件](#规则java文件)和[描述性HTML和JSON文件](#描述性html和json文件)是必须的，[测试文件](#测试文件)和[规则示例文件](#规则示例文件可选若写测试则必需)可选但强烈推荐。

### 插件项目结构

SonarQube的Java自定义规则是通过插件来实现的。插件是一个完整的maven项目，项目结构如下：

```
插件名
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── package路径  // 如com.example.java
    │   │       ├── checks  // 放入规则文件
    │   │       │   ├── 规则Java文件1.java
    │   │       │   ├── 规则Java文件2.java
    │   │       │   └── ...
    │   │       ├── MyJavaFileCheckRegistrar.java
    │   │       ├── MyJavaRulesDefinition.java
    │   │       ├── MyJavaRulesPlugin.java
    │   │       ├── package-info.java
    │   │       └── RulesList.java
    │   └── resources
    │       └── org.sonar.l10n.java.rules.java  // 这个路径是写死的，不要改动
    │           ├── 规则Java文件1.html
    │           ├── 规则Java文件1.json
    │           ├── 规则Java文件2.html
    │           ├── 规则Java文件2.json
    │           └── ...  // 每一个规则都有一个html和json文件
    └── test  // 可选
        ├── files
        │   ├── 规则示例文件1.java
        │   ├── 规则示例文件2.java
        │   └── ...
        └── java
            └── package路径  // 如com.example.java
                ├── checks  // 放入测试文件
                │   ├── 规则Java文件1Test.java
                │   ├── 规则Java文件2Test.java
                │   └── ...
                ├── MyJavaFileCheckRegistrarTest.java
                ├── MyJavaRulesDefinitionTest.java
                └── MyJavaRulesPluginTest.java
```

每新增一个规则，都需要在`/src/main/java/package路径/RulesList.java`的`getJavaChecks()`方法或`getJavaTestChecks()`方法中添加这个规则的类。如果有测试，则同时需要在`/src/test/java/package路径/MyJavaFileCheckRegistrarTest.java`的`checkNumberRules()`方法中修改规则的总数量。

完成规则构建后，用`mvn clean package`命令打包，把生成的jar包放到SonarQube的`$SONAR_HOME/extensions/plugins`目录下，重启SonarQube即可。



### 规则Java文件

#### `IssuableSubscriptionVisitor`类

如果只需要检测少量几种语法树，而且不同语法树之间互不影响，则可以用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)，这个类只需要实现`visitNode`和`nodesToVisit`方法即可。

```java
package package路径.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.Collections;
import java.util.List;

@Rule(key = "自定义规则编码，用下面的类名即可")
public class 自定义规则类 extends IssuableSubscriptionVisitor {

  // 需要检查哪些类型的语法树
  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Arrays.asList(语法树种类，例如Tree.Kind.METHOD，可输入多个);
  }

  // 对单个语法树进行检测（比如单个方法、类等等）
  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    // 我们知道这个语法树的类型，所以可以强制转换；如果有多个类型，可以用tree.is(Kind kind)或者instanceof判断
    MethodTree methodTree = (MethodTree) tree;
    // 做一系列的检查，如果检查到问题，就调用reportIssue方法
    if (methodTree.parameters().size() == 1) {
      Symbol.MethodSymbol methodSymbol = methodTree.symbol();
      Type firstParamType = methodSymbol.parameterTypes().get(0);
      Type returnType = methodSymbol.returnType().type();
      if (returnType.is(firstParamType.fullyQualifiedName())) {
        // 第一个参数tree是需要报错的节点
        this.reportIssue(tree, "Parameter type must be different from return type");
      }
    }
  }

  @Override
  public void leaveNode(Tree tree) {
    // 如果需要在检测完语法树后再做一些操作，可以在这里写
  }

  // 指定扫描文件的方式（一般不会重载这个方法，使用默认的方法即可）
  @Override
  public void scanFile(JavaFileScannerContext context) {
    // 自定义扫描方式
  }

}
```


#### `BaseTreeVisitor`类

如果需要更复杂的检测，则可以用[`BaseTreeVisitor`](#basetreevisitor类)和`JavaFileScanner`，这个类需要实现`scanFile`方法并按需重载一些检测语法树的方法。注意使用这个类报告问题时，`reportIssue()`是由`JavaFileScannerContext`调用的，而不是[`BaseTreeVisitor`](#basetreevisitor类)。

```java
package package路径.checks;

import org.sonar.check.Rule;
import org.sonar.check.RuleProperty;
import org.sonar.plugins.java.api.JavaFileScanner;
import org.sonar.plugins.java.api.JavaFileScannerContext;
import org.sonar.plugins.java.api.tree.BaseTreeVisitor;
import org.sonar.plugins.java.api.tree.ClassTree;
import org.sonar.plugins.java.api.tree.MethodTree;

import java.util.List;

@Rule(key = "自定义规则编码")
public class 自定义规则类 extends BaseTreeVisitor implements JavaFileScanner {

  // 这个变量必须有
  private JavaFileScannerContext context;

  // 如果规则有一些属性，可以在这里定义
  @RuleProperty(defaultValue = ..., description = ...)
  protected ...;

  // 指定扫描文件的方式（必须有），一般就写如下的方法
  @Override
  public void scanFile(JavaFileScannerContext context) {
    this.context = context;
    this.scan(context.getTree());
  }

  // 检测方法的语法树
  @Override
  public void visitMethod(MethodTree tree) {
    // 对方法的语法树进行检测；在这里检测是前序遍历
    // 在检测过程中，如果检测到问题，就调用reportIssue方法
    // 第二个参数tree是需要报错的节点
    this.context.reportIssue(this, tree, "Avoid declaring methods (don't ask why)");

    // 检测完成后，调用super的方法，继续检测方法内部的语法树
    super.visitMethod(tree);

    // 如果需要先检测方法内部的语法树，再检测方法的语法树，则把对语法树的检测放在这里，实现后序遍历
  }

  // 检测类的语法树
  @Override
  public void visitClass(ClassTree tree) {
    // ...
  }

  // 还可以检测语句、变量、数组等语法树
  // 例如visitAssertStatement、visitWhileStatement、visitTypeCast、visitAnnotation等等
}
```



### 描述性HTML和JSON文件

#### HTML

HTML文件是这个规则在SonarQube管理页面中展示的内容。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>规则名称</title>
  </head>
  <body>
    <p>规则描述</p>

    <h2>Noncompliant Code Example</h2>
    <pre>
    给一个不符合规则的代码的例子
    </pre>

    <h2>Compliant Solution</h2>
    <pre>
    给一个符合规则的代码的例子
    </pre>
  </body>
</html>
```


#### JSON

JSON文件描述这个规则的属性，比如类型、标签、严重性等等。

```json
{
  "title": 规则标题或名称,
  "type": 规则类型，可选"BUG"、"VULNERABILITY"、"CODE_SMELL"、"SECURITY_HOTSPOT",
  "status": 规则状态，可选"ready"、"beta"、"deprecated"、"removed"、"experimental"、"external"、"test"、"changed",
  "remediation": {
    "func": 修复类型,
    "constantCost": 预计修复耗时
  },
  "tags": [
    规则标签
  ],
  "defaultSeverity": 严重性，可选"Minor"、"Major"、"Critical"、"Blocker",
}
```



### 规则示例文件（可选，若写测试则必需）

一个简单的示例文件，用于测试规则，里面会包含符合规则的代码和不符合规则的代码。

对于不符合规则的代码，需要在不符合规则的地方用注释`// Noncompliant`标记出来。这个注释必须跟随在代码之后。如果有多个地方不符合规则，则在每个地方都要标记。一个示例文件中至少要有一处不符合规则的代码。

符合规则的代码则不需要标记。

```java
class 随便取类名 {
  int     foo3(int value) { return 0; } // Noncompliant {{解释不符合规则的原因（可选）}}
  Object  foo4(int value) { return null; }
  MyClass foo5(MyClass value) {return null; } // Noncompliant
  int     foo6(int value, String name) { return 0; }
  int     foo7(int ... values) { return 0;}
}
```

如果错误地标记了行，则测试时会报错`java.lang.AssertionError`。



### 测试文件（可选）

一个测试文件可以测试多个[规则示例文件](#规则示例文件)。

```java
package package路径.checks;

import org.junit.jupiter.api.Test;
import org.sonar.java.checks.verifier.CheckVerifier;

public class 测试文件名 {

  @Test
  void 测试方法名() {
    CheckVerifier.newVerifier()
      .onFile("src/test/files/规则示例文件.java")
      .withCheck(new 规则类())
      .verifyIssues();
  }

  @Test
  void 测试方法名2() {
    规则类 rule = new 规则类();
    // 可以设置规则的属性
    rule.name = ...;

    CheckVerifier.newVerifier()
      .onFile("src/test/files/规则示例文件.java")
      .withCheck(rule)
      .verifyIssues();
  }

}
```




## SonarQube自定义规则示例

以下的规则仅做示例用，规则本身不一定有实际用途。

#### \[[Java开发手册](https://github.com/alibaba/p3c)\] 1.1.1 【强制】代码中的命名均不能以下划线或美元符号开始，也不能以下划线或美元符号结束。

由于命名规范多用于类、方法和变量的命名，这里考虑遍历这三者的语法树。这里需要遍历多种语法树，但语法树之间互不影响，所以[`BaseTreeVisitor`](#basetreevisitor类)和[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)都可以用。

使用[`BaseTreeVisitor`](#basetreevisitor类)的规则文件：

使用到的类和方法：

* [`BaseTreeVisitor`](#basetreevisitor类)：`visitClass()`、`visitMethod()`、`visitVariable()`

* `JavaFileScannerContext`：`reportIssue()`、`getTree()`

* [`ClassTree`](#类树classtree)：`simpleName()`

* [`MethodTree`](#方法树methodtree)：`simpleName()`

* [`VariableTree`](#变量树variabletree)：`simpleName()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.JavaFileScanner;
import org.sonar.plugins.java.api.JavaFileScannerContext;
import org.sonar.plugins.java.api.tree.*;

import java.util.Objects;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends BaseTreeVisitor implements JavaFileScanner {

  private JavaFileScannerContext context;

  @Override
  public void scanFile(JavaFileScannerContext context) {
    this.context = context;
    this.scan(context.getTree());
  }

  @Override
  public void visitClass(ClassTree tree) {
    String className = Objects.requireNonNull(tree.simpleName()).name();
    this.checkIssue(tree, className);
    super.visitClass(tree);
  }

  @Override
  public void visitMethod(MethodTree tree) {
    String methodName = Objects.requireNonNull(tree.simpleName().name());
    this.checkIssue(tree, methodName);
    super.visitMethod(tree);
  }

  @Override
  public void visitVariable(VariableTree tree) {
    String variableName = Objects.requireNonNull(tree.simpleName().name());
    this.checkIssue(tree, variableName);
    super.visitVariable(tree);
  }
  
  private void checkIssue(Tree tree, String name) {
    if (name.startsWith("_") || name.startsWith("$") || name.endsWith("_") || name.endsWith("$")) {
      this.context.reportIssue(
        this,
        tree,
        String.format("Variable identifier %s starts or ends with underscore or dollar sign", name)
      );
    }
  }

}
```

使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)的规则文件：

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`ClassTree`](#类树classtree)：`simpleName()`

* [`MethodTree`](#方法树methodtree)：`simpleName()`

* [`VariableTree`](#变量树variabletree)：`simpleName()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.ClassTree;
import org.sonar.plugins.java.api.tree.MethodTree;
import org.sonar.plugins.java.api.tree.Tree;
import org.sonar.plugins.java.api.tree.VariableTree;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Arrays.asList(Tree.Kind.CLASS, Tree.Kind.METHOD, Tree.Kind.VARIABLE);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    String name;
    if (tree instanceof ClassTree) {
      ClassTree classTree = (ClassTree) tree;
      name = Objects.requireNonNull(classTree.simpleName()).name();

    } else if (tree instanceof MethodTree) {
      MethodTree methodTree = (MethodTree) tree;
      name = Objects.requireNonNull(methodTree.simpleName()).name();
    } else {
      VariableTree variableTree = (VariableTree) tree;
      name = Objects.requireNonNull(variableTree.simpleName()).name();
    }
    if (name.startsWith("_") || name.startsWith("$") || name.endsWith("_") || name.endsWith("$")) {
      this.context.reportIssue(
        this,
        tree,
        String.format("Identifier %s starts or ends with underscore or dollar sign", name)
      );
    }
  }
}
```


#### \[[Java开发手册](https://github.com/alibaba/p3c)\] 1.1.6 【强制】常量命名应该全部大写，单词间用下划线隔开，力求语义表达完整清楚，不要嫌名字长。

检测单词间是否用下划线隔开比较困难，需要识别单个单词，这里只检测常量（即`final`修饰的变量）命名是否全部大写。由于只检测变量，这里选择使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`VariableTree`](#变量树variabletree)：`simpleName()`、`modifiers()`

* [`ModifiersTree`](#多修饰符树modifierstree)：`modifiers()`

* [`ModifierKeywordTree`](#修饰符关键字树modifierkeywordtree)：`modifier()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Collections.singletonList(Tree.Kind.VARIABLE);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    VariableTree variableTree = (VariableTree) tree;
    String name = Objects.requireNonNull(variableTree.simpleName()).name();
    List<Modifier> modifierTrees = variableTree.modifiers()
      .modifiers()
      .stream()
      .map(ModifierKeywordTree::modifier)
      .collect(Collectors.toList());
    if (modifierTrees.contains(Modifier.FINAL) && !name.toUpperCase().equals(name)) {
      this.context.reportIssue(
        this,
        tree,
        String.format("Final variable %s's identifier must be in upper case", name)
      );
    }

  }
}
```


#### \[[Java开发手册](https://github.com/alibaba/p3c)\] 1.1.11 【强制】避免在子父类的成员变量之间、或者不同代码块的局部变量之间采用完全相同的命名，使可理解性降低。

由于只遍历类树，这里选择使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)。

注意SonarJava并不支持查找类的子类，查找父类时也仅支持取得父类的类型（即类名），所以需要自己实现子类的父类的对应关系。这里选择使用两个`Map`，`CLASS_FIELDS`存储一个类的名字和它的所有类变量名，`PARENT_TO_CHILDREN`存储一个类的名字和它的所有子类的名字。

如果需要测试这个规则，在规则示例中，`// Noncompliant`注解需要写在类定义的后面，因为这里是在类树中`reportIssue()`。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`ClassTree`](#类树classtree)：`symbol()`、`members()`、`superClass()`

* [`VariableTree`](#变量树variabletree)：`simpleName()`

* [`TypeTree`](#类型树typetree)：`symbolType()`

* [`Symbol`](#符号symbol)：`name()`

* [`Type`](#类型type)：`name()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.*;
import java.util.stream.Collectors;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  private final Map<String, Set<String>> CLASS_FIELDS;

  private final Map<String, List<String>> PARENT_TO_CHILDREN;

  public JavaDevRuleCheck() {
    super();
    this.CLASS_FIELDS = new HashMap<>();
    this.PARENT_TO_CHILDREN = new HashMap<>();
  }

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Collections.singletonList(Tree.Kind.CLASS);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    ClassTree classTree = (ClassTree) tree;
    // 取得所有的类变量
    Set<String> fields = classTree.members()
      .stream()
      .filter(member -> member.is(Tree.Kind.VARIABLE))
      .map(member -> {
        VariableTree variableTree = (VariableTree) member;
        return variableTree.simpleName().name();
      })
      .collect(Collectors.toSet());
    // 取得类名
    String className = classTree.symbol().name();
    // 把类名和类变量的映射放入classFields
    this.CLASS_FIELDS.put(className, fields);
    // 如果这个类有子类，检查是否与任何一个子类有重复的变量
    // 如果parentToChild中有这个类，说明至少一个子类是被检查过了的
    if (this.PARENT_TO_CHILDREN.containsKey(className)) {
      for (String childClassName: this.PARENT_TO_CHILDREN.get(className)) {
        Set<String> childFields = this.CLASS_FIELDS.get(childClassName);
        Set<String> commonFields = new HashSet<>(fields);
        commonFields.retainAll(childFields);
        if (!commonFields.isEmpty()) {
          this.context.reportIssue(
            this,
            classTree,
            String.format("Parent class %s and child class %s contain duplicate fields", className, childClassName)
          );
        }
      }
    }
    // 如果这个类有父类，把它的父类和它的关系存入parentToChild
    if (classTree.superClass() != null) {
      String parentClassName = classTree.superClass().symbolType().name();
      if (this.PARENT_TO_CHILDREN.containsKey(parentClassName)) {
        this.PARENT_TO_CHILDREN.get(parentClassName).add(className);
      } else {
        List<String> children = new ArrayList<>();
        children.add(className);
        this.PARENT_TO_CHILDREN.put(parentClassName, children);
      }
      // 如果父类在classFields里面，则检查是否和父类有重复的变量
      Set<String> parentFields = this.CLASS_FIELDS.get(parentClassName);
      Set<String> commonFields = new HashSet<>(fields);
      commonFields.retainAll(parentFields);
      if (!commonFields.isEmpty()) {
        this.context.reportIssue(
          this,
          classTree,
          String.format("Parent class %s and child class %s contain duplicate fields", parentClassName, className)
        );
      }
    }
  }
}
```


#### \[[Java开发手册](https://github.com/alibaba/p3c)\] 1.2.2 【强制】long 或 Long 赋值时，数值后使用大写 L，不能是小写 l，小写容易跟数字混淆，造成误解。

因为我们只关心`long`类型的字面量，所以这里选择使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)，并把`nodesToVisit()`中的类型列表设为`Tree.Kind.LONG_LITERAL`。这样只会对`long`类型的字面量进行检测。

如果使用[`BaseTreeVisitor`](#basetreevisitor类)，则需要自行在`visitLiteral()`方法中判断字面量的类型，如果是`long`类型，则进行检测。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`LiteralTree`](#字面量树literaltree)：`value()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.*;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Collections.singletonList(Tree.Kind.LONG_LITERAL);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    LiteralTree literalTree = (LiteralTree) tree;
    if (literalTree.value().endsWith("l")) {
      this.reportIssue(tree, "Long literal should end with \"L\", not \"l\"");
    }
  }
}
```


#### \[[Java开发手册](https://github.com/alibaba/p3c)\] 1.3.2 【强制】左小括号和右边相邻字符之间不需要空格；右小括号和左边相邻字符之间也不需要空格；而左大括号前需要加空格。

此处仅检查小括号的使用，对大括号的检查是类似的。

这个规则比较麻烦。Java中的括号会出现在不同地方，比如`while`语句、`for`循环、数学运算表达式等等。为了让SonarQube能够检测所有的这些情况，我们需要列出所有小括号可能出现的地方，然后在这些地方检测小括号的使用。因为需要检测多种语句，所以这里选择使用[`BaseTreeVisitor`](#basetreevisitor类)。

规则的核心思想是取得左右小括号的[语法token](#语法tokensyntaxtoken)，再取得括号中的语句的第一个和最后一个[语法token](#语法tokensyntaxtoken)，然后比较这些token的位置。

由于完整的规则文件过长，这里仅展示部分方法，其它方法的构造是类似的。

使用到的类和方法：

* [`BaseTreeVisitor`](#basetreevisitor类)：`visitLambdaExpression()`、`visitParenthesized()`、`visitTypeCast()`、`visitDoWhileStatement()`、`visitForEachStatement()`、`visitForStatement()`、`visitIfStatement()`、`visitSynchronizedStatement()`、`visitTryStatement()`

* [`SyntaxToken`](#语法tokensyntaxtoken)：`column()`、`identifierToken()`

* [`Tree`](#树tree)：`firstToken()`、`lastToken()`

* [`LambdaExpressionTree`](#lambda表达式树lambdaexpressiontree)：`openParenToken()`、`closeParenToken()`、`parameters()`

* [`VariableTree`](#变量树variabletree)：`simpleName()`

* [`ParenthesizedTree`](#括号树parenthesizedtree)：`openParenToken()`、`closeParenToken`、`expression()`

* [`TypeCastTree`](#类型转换树typecasttree)：`openParenToken()`、`closeParenToken()`、`type()`

* [`DoWhileStatementTree`](#do-while语句树dowhilestatementtree)：`openParenToken()`、`closeParenToken()`、`condition()`

* [`ForEachStatement`](#foreach语句树foreachstatement)：`openParenToken()`、`closeParenToken()`、`variable()`、`expression()`

* [`ForStatementTree`](#for语句树forstatementtree)：`openParenToken()`、`closeParenToken()`、`initializer()`、`update()`

* [`IfStatementTree`](#if语句树ifstatementtree)：`openParenToken()`、`closeParenToken()`、`condition()`

* [`SynchronizedStatementTree`](#synchronized语句树synchronizedstatementtree)：`openParenToken()`、`closeParenToken()`、`expression()`

* [`TryStatementTree`](#try语句树trystatementtree)：`openParenToken()`、`closeParenToken()`、`resourceList()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.JavaFileScanner;
import org.sonar.plugins.java.api.JavaFileScannerContext;
import org.sonar.plugins.java.api.tree.*;

import java.util.List;
import java.util.Objects;

@Rule(key = "JavaDevRuleCheckBaseVisitor")
public class JavaDevRuleCheckBaseVisitor extends BaseTreeVisitor implements JavaFileScanner {

  private JavaFileScannerContext context;

  @Override
  public void scanFile(JavaFileScannerContext context) {
    this.context = context;
    this.scan(context.getTree());
  }

  @Override
  public void visitLambdaExpression(LambdaExpressionTree tree) {
    if (tree.openParenToken() != null) {
      int openParenCol = Objects.requireNonNull(tree.openParenToken()).column();
      int closeParenCol = Objects.requireNonNull(tree.closeParenToken()).column();
      List<VariableTree> parameters = tree.parameters();
      if (parameters.isEmpty()) {
        if (openParenCol + 1 != closeParenCol) {
          this.context.reportIssue(this, tree, "No space inside empty parentheses");
        }
      } else {
        SyntaxToken firstVarToken = parameters.get(0).simpleName().identifierToken();
        SyntaxToken lastVarToken = parameters.get(parameters.size()-1).simpleName().identifierToken();
        this.reportIssue(firstVarToken, lastVarToken, openParenCol, closeParenCol, tree);
      }
    }
    super.visitLambdaExpression(tree);
  }

  @Override
  public void visitParenthesized(ParenthesizedTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.expression().firstToken();
    SyntaxToken lastToken = tree.expression().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitParenthesized(tree);
  }

  @Override
  public void visitTypeCast(TypeCastTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.type().firstToken();
    SyntaxToken lastToken = tree.type().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitTypeCast(tree);
  }

  @Override
  public void visitDoWhileStatement(DoWhileStatementTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.condition().firstToken();
    SyntaxToken lastToken = tree.condition().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitDoWhileStatement(tree);
  }

  @Override
  public void visitForEachStatement(ForEachStatement tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.variable().firstToken();
    SyntaxToken lastToken = tree.expression().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitForEachStatement(tree);
  }

  @Override
  public void visitForStatement(ForStatementTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.initializer().firstToken();
    SyntaxToken lastToken = tree.update().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitForStatement(tree);
  }

  @Override
  public void visitIfStatement(IfStatementTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.condition().firstToken();
    SyntaxToken lastToken = tree.condition().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitIfStatement(tree);
  }

  @Override
  public void visitSynchronizedStatement(SynchronizedStatementTree tree) {
    int openParenCol = tree.openParenToken().column();
    int closeParenCol = tree.closeParenToken().column();
    SyntaxToken firstToken = tree.expression().firstToken();
    SyntaxToken lastToken = tree.expression().lastToken();
    this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    super.visitSynchronizedStatement(tree);
  }

  @Override
  public void visitTryStatement(TryStatementTree tree) {
    if (tree.openParenToken() != null) {
      int openParenCol = Objects.requireNonNull(tree.openParenToken()).column();
      int closeParenCol = Objects.requireNonNull(tree.closeParenToken()).column();
      SyntaxToken firstToken = tree.resourceList().firstToken();
      SyntaxToken lastToken = tree.resourceList().lastToken();
      this.reportIssue(firstToken, lastToken, openParenCol, closeParenCol, tree);
    }
    super.visitTryStatement(tree);
  }

  private void reportIssue(SyntaxToken firstToken, SyntaxToken lastToken, int openParenCol, int closeParenCol, Tree tree) {
    if (firstToken != null && firstToken.column() > openParenCol + 1) {
      this.context.reportIssue(this, tree, "No space between open parenthesis and expression");
    } else if (lastToken != null && lastToken.column() + lastToken.text().length() < closeParenCol) {
      this.context.reportIssue(this, tree, "No space between closing parenthesis and expression");
    }
  }

}
```


#### `for`循环和`for-each`循环里不能嵌套`if`。

由于[`ForStatementTree`](#for语句树forstatementtree)和[`ForEachStatement`](#for-each语句foreachstatement)都会把循环里的代码块当作一整个语句来处理（而不是当作代码块），我们需要自己定义一个树的遍历器去检查循环里的代码块的内容。

检查的逻辑是：只检查循环里的代码块，在循环里只要遇到`if`语句，不论在哪里遇到都报告错误。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`BaseTreeVisitor`](#basetreevisitor类)：`visitIfStatement()`

* [`Tree`](#树tree)：`accept()`

* [`ForStatementTree`](#for语句树forstatementtree)：`statement()`

* [`ForEachStatement`](#for-each语句foreachstatement)：`statement()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.*;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Arrays.asList(Tree.Kind.FOR_STATEMENT, Tree.Kind.FOR_EACH_STATEMENT);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    // ForStatementTree和ForEachStatement都有statement()方法，
    // 所以可以定义一个统一的StatementTree变量来代表循环里的代码块
    StatementTree codeBlock;
    // tree只能是ForStatementTree或者ForEachStatement，这里用is()判断种类
    if (tree.is(Tree.Kind.FOR_STATEMENT)) {
      ForStatementTree forTree = (ForStatementTree) tree;
      codeBlock = forTree.statement();
    } else {
      ForEachStatement forEachTree = (ForEachStatement) tree;
      codeBlock = forEachTree.statement();
    }
    // 手动放入visitor来自定义节点访问顺序，visitor只能检查codeBlock节点和它的所有子节点
    codeBlock.accept(new StatementVisitor());
  }

  // 自定义一个检查循环里的代码的visitor
  private class StatementVisitor extends BaseTreeVisitor {
    // 因为只会检查循环里的代码，所以只要遇到if语句就报告错误。循环外的if语句不会被检查
    @Override
    public void visitIfStatement(@ParametersAreNonnullByDefault IfStatementTree tree) {
      JavaDevRuleCheck.this.reportIssue(tree, "Should not nest if statement inside for loop");
      // （可选）如果if里有嵌套的语句，也要检查
      super.visitIfStatement(tree);
    }
  }
}
```


#### `final`修饰的变量不能被重新赋值。

由于我们是检查**变量**是否被`final`修饰以及是否被重新赋值，我们只会遍历[变量树](#变量树variabletree)（可能还会遍历[赋值表达式树](#赋值表达式树assignmentexpressiontree)）所以这里选择使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)。

检查的逻辑是：对于每一个`final`变量，检查它是否在声明时就已经被赋值了，如果没有，则允许一次赋值（`allowedAssignments`设为`1`），否则一次赋值都不允许（`allowedAssignments`设为`0`）。然后对于每一个使用了这个变量的地方，检查是否为赋值语句。如果`final`变量是被赋值的一方，那么赋值次数`assignments`就加一；如果`assignments`大于允许的赋值次数（`allowedAssignments`），则报告错误。

这里的诀窍是，如果这个`final`变量是被赋值的一方，**它的[标识符](#标识符树identifiertree)的父节点一定是[赋值表达式树](#赋值表达式树assignmentexpressiontree)。**如果是赋值给别的变量，则它的父节点会是别的树，比如[变量树](#变量树variabletree)、[表达式树](#表达式树expressiontree)等。通过这个方法可以方便地检验`final`变量是否被重新赋值。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`nodesToVisit()`、`reportIssue()`

* [`Tree`](#树tree)：`parent()`

* [`VariableTree`](#变量树variabletree)：`modifiers()`、`symbol()`、`initializer()`

* [`ModifiersTree`](#多修饰符树modifierstree)：`modifiers()`

* [`ModifierKeywordTree`](#修饰符关键字树modifierkeywordtree)：`modifier()`

* [`Symbol`](#符号symbol)：`usages()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.semantic.Symbol;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.ParametersAreNonnullByDefault;
import java.util.*;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  @Override
  public List<Tree.Kind> nodesToVisit() {
    return Collections.singletonList(Tree.Kind.VARIABLE);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    VariableTree variableTree = (VariableTree) tree;
    // 取得变量的修饰符
    List<ModifierKeywordTree> modifiers = variableTree.modifiers().modifiers();
    for (ModifierKeywordTree modifier: modifiers) {
      // 如果变量的修饰符中有final
      if (modifier.modifier() == Modifier.FINAL) {
        // 先检查final变量是否在声明时就已经被赋值了
        // 如果没有，则应该允许第一次赋值
        int allowedAssignments = variableTree.initializer() == null ? 1 : 0;
        int assignments = 0;
        // 对每一个使用了这个变量的地方进行检查
        for (IdentifierTree identifierTree: variableTree.symbol().usages()) {
          // 如果这个变量被重新赋值了
          if (identifierTree.parent() instanceof AssignmentExpressionTree) {
            assignments ++;
            if (assignments > allowedAssignments) {
              this.reportIssue(identifierTree, "Cannot reassign final variables");
            }
          }
        }
        // 修饰符中只会有一个final，所以检查到final就可以停止了
        break;
      }
    }
  }
}
```


#### 不能出现没有在文件中使用的`import`语句。

由于我们需要遍历整个文件，所以这里选择使用[`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)。

这个例子中使用了SonarJava的一个工具类`ExpressionsHelper`来取得完整的`import`语句后的类名，源代码在[这里](https://github.com/SonarSource/sonar-java/blob/master/java-checks/src/main/java/org/sonar/java/checks/helpers/ExpressionsHelper.java)。

使用到的类和方法：

* [`IssuableSubscriptionVisitor`](#issuablesubscriptionvisitor类)：`visitNode()`、`leaveNode()`、`nodesToVisit()`、`reportIssue()`

* [`Tree`](#树tree)：`accept()`

* [`CompilationUnitTree`](#编译单元树compilationunittree)：`imports()`

* [`ImportTree`](#import语句树importtree)：`is()`、`qualifiedIdentifier()`

* [`VariableTree`](#变量树variabletree)：`type()`

* [`TypeTree`](#类型树typetree)：`symbolType()`

* [`Type`](#类型type)：`fullyQualifiedName()`

* [`NewClassTree`](#新建类对象树newclasstree)：`symbolType()`

* `ExpressionsHelper`：`concatenate()`

```java
package org.sonar.samples.java.checks;

import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.*;

import javax.annotation.Nullable;
import javax.annotation.ParametersAreNonnullByDefault;
import java.util.*;

@Rule(key = "JavaDevRuleCheck")
public class JavaDevRuleCheck extends IssuableSubscriptionVisitor {

  // 用这个HashMap来存import的类和它对应的ImportTree，ImportTree用于寻找报错位置
  private final Map<String, ImportTree> imports = new HashMap<>();

  @Override
  public List<Tree.Kind> nodesToVisit() {
    // 需要从整个文件的树入手，所以选择CompilationUnitTree
    return Collections.singletonList(Tree.Kind.COMPILATION_UNIT);
  }

  @Override
  public void visitNode(@ParametersAreNonnullByDefault Tree tree) {
    CompilationUnitTree compilationUnitTree = (CompilationUnitTree) tree;
    // 取得所有的import语句树并加入imports中
    compilationUnitTree.imports()
      .stream()
      .filter(importTree -> importTree.is(Tree.Kind.IMPORT))
      .map(ImportTree.class::cast)
      .forEach(importTree -> {
        // 取得完整的import语句后的类名
        String importName = ExpressionsHelper.concatenate((ExpressionTree) importTree.qualifiedIdentifier());
        this.imports.put(importName, importTree);
      });
    tree.accept(new TypeVisitor());
  }

  // 在遍历完整个文件后，对每个没有使用到的import语句报错
  @Override
  public void leaveNode(@ParametersAreNonnullByDefault Tree tree) {
    this.imports.forEach((unusedImport, syntaxNode) -> {
      this.reportIssue(syntaxNode, "Remove unused import " + unusedImport);
    });
  }

  // 自定义一个visitor来检查变量的类型
  private class TypeVisitor extends BaseTreeVisitor {
    // 检查新建类实例时使用的类的类型
    @Override
    public void visitNewClass(NewClassTree tree) {
      String typeName = tree.symbolType().fullyQualifiedName();
      JavaDevRuleCheck.this.imports.remove(typeName);
      super.visitNewClass(tree);
    }

    // 检查所有变量的类型
    @Override
    public void visitVariable(VariableTree tree) {
      String typeName = tree.type().symbolType().fullyQualifiedName();
      JavaDevRuleCheck.this.imports.remove(typeName);
      super.visitVariable(tree);
    }
  }

}
```