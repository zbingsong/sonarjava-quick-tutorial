# SonarQube自定义规则简便文档

Sonar通过遍历语法树的形式来进行代码检查。

官方的sonar-java插件里已经内置了很多规则，可以在[这里](https://github.com/SonarSource/sonar-java/tree/master/java-checks/src/main/java/org/sonar/java/checks)看到源码。




## 语法树组件

### 树`Tree`

整个文件的语法树中的一个节点，表示一个语句、一个表达式、一个变量、一个方法、一个类等等。树的类型非常多，具体可以在`org.sonar.plugins.java.api.tree.BaseTreeVisitor`中找到。

所有的树类型的组件都可以使用`Tree`类的方法。

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

* `Arguments arguments()`：获取注解的参数。

  `Arguments`是[`ListTree`](#列表树listtree)的子类，元素类型为[`ExpressionTree`](#表达式树expressiontree)。有获取左右括号的[`SyntaxToken`](#语法tokensyntaxtoken)的方法`openParenToken()`和`closeParenToken()`。


#### 数组元素访问表达式树`ArrayAccessExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

通过下标访问数组的元素的表达式，例如`arr[1]`。

* `ExpressionTree expression()`：获取数组访问的[表达式](#表达式树expressiontree)。

* `ArrayDimensionTree dimension()`：获取数组的[长度](#arraydimensiontree)。


#### 赋值表达式树`AssignmentExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 

* `ExpressionTree variable()`：获取赋值的左边的[表达式](#表达式树expressiontree)。

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


##### 常量树`LiteralTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `String value()`：获取常量的值。

* `SyntaxToken token()`：获取常量的[语法token](#语法tokensyntaxtoken)。


#### 成员访问树`MemberSelectExpressionTree`

父类：[`ExpressionTree`](#表达式树expressiontree) 、[`TypeTree`](#类型树typetree)

通过`.`访问成员的表达式，例如`System.out.println()`、`Arrays.asList()`、`classInstance.field`。

注意如果有连续的`.`，那么每个`.`都会有一个`MemberSelectExpressionTree`。例如`java.lang.annotation.*`会被解析为三个`MemberSelectExpressionTree`，分别是`java.lang.annotation.*`、`java.lang.annotation`和`java.lang`。

* `ExpressionTree expression()`：获取整个访问成员的[表达式](#表达式树expressiontree)。

* `SyntaxToken operatorToken()`：获取`.`的[语法token](#语法tokensyntaxtoken)。

* `IdentifierTree identifier()`：获取被访问的成员的[标识符树](#标识符树identifiertree)。


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

父类：[`ExpressionTree`](#表达式树expressiontree)

* `@Nullable TypeTree type()`：获取数组的[类型树](#类型树typetree)。

* `@Nullable SyntaxToken newKeyword()`：获取`new`关键字的[语法token](#语法tokensyntaxtoken)。

* `List<ArrayDimensionTree> dimensions()`：获取数组的[长度](#数组长度树arraydimensiontree)。

* `@Nullable SyntaxToken openBraceToken()`：获取左大括号的[语法token](#语法tokensyntaxtoken)。

* `ListTree<ExpressionTree> initializers()`：获取数组的初始值，以[列表树](#列表树listtree)的方式列出。

* `@Nullable SyntaxToken closeBraceToken()`：获取右大括号的[语法token](#语法tokensyntaxtoken)。


#### 新建类树`NewClassTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `@Nullable ExpressionTree enclosingExpression()`：获取外部类的[表达式](#表达式树expressiontree)。仅在这个新建类是内部类时才会有。

* `@Nullable SyntaxToken dotToken()`：获取`.`的[语法token](#语法tokensyntaxtoken)。仅在这个新建类是内部类时才会有。

* `@Nullable SyntaxToken newKeyword()`：获取`new`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable TypeArguments typeArguments()`：获取类的[泛型参数值列表](#泛型参数值列表typearguments)。

* `TypeTree identifier()`：获取类的[类型树](#类型树typetree)。

* `Arguments arguments()`：获取新建类时使用的[参数列表](#参数列表arguments)。

* `@Nullable ClassTree classBody()`：获取类的内容（[类树](#类树classtree)）。

* `Symbol constructorSymbol()`：获取类的`constructor`的[符号](#符号symbol)。


#### 括号树`ParenthesizedTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

一个被括号包起来的表达式，例如`(1 + 2) * 3`中的`(1 + 2)`。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取括号中的[表达式](#表达式树expressiontree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。


#### 类型转换树`TypeCastTree`

父类：[`ExpressionTree`](#表达式树expressiontree)

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `TypeTree type()`：获取转换的[类型树](#类型树typetree)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree expression()`：获取转换的[表达式](#表达式树expressiontree)。

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

  _为什么不用`BlockTree`？_

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

  _为什么不用`BlockTree`？_


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

  _为什么不用`BlockTree`？_


#### If语句树`IfStatementTree`

父类：[`StatementTree`](#语句树statementtree) 

* `SyntaxToken ifKeyword()`：获取if关键字的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken openParenToken()`：获取左括号的[语法token](#语法tokensyntaxtoken)。

* `SyntaxToken closeParenToken()`：获取右括号的[语法token](#语法tokensyntaxtoken)。

* `ExpressionTree condition()`：获取if语句的条件[表达式](#表达式树expressiontree)。

* `StatementTree thenStatement()`：获取if语句的代码块（即如果if条件为真时会执行的语句）。

  _不理解为何这个方法返回的是`StatementTree`而不是`BlockTree`，很明显if语句后的代码块是一个块而不是一个语句_

* `@Nullable SyntaxToken elseKeyword()`：获取else关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable StatementTree elseStatement()`：获取else语句的代码块（即如果if条件为假时会执行的语句）。

  _不理解为何这个方法返回的是`StatementTree`而不是`BlockTree`，很明显else语句后的代码块是一个块而不是一个语句_


#### Labeled语句树`LabeledStatementTree`

父类：[`StatementTree`](#语句树statementtree)

* `IdentifierTree label()`：获取标签的[标识符树](#标识符树identifiertree)。

* `SyntaxToken colonToken()`：获取冒号的[语法token](#语法tokensyntaxtoken)。

* `StatementTree statement()`：获取标签后的语句。

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

* `StatementTree statement()`：获取`while`语句后的代码块。

  _为什么不用`BlockTree`？_


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

* `IdentifierTree.name()`：标识符的名称。

* `IdentifierTree.symbol()`：标识符的[符号](#符号symbol)。


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

父类：[`ListTree`](#列表树listtree)（元素类型为[修饰符树](#修饰符树`ModifierTree`)）

表示一系列的修饰符。

`ModifiersTree`是`ListTree`的子类，可以直接当作一个包含`ModifierTree`的`List`来使用。

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

模组`module`是**Java 9**中的新特性，用于模块化Java程序。详见[Java 9模组说明文章](https://www.oracle.com/corporate/features/understanding-java-9-modules.html)。


#### `RequiresDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)


#### `ExportsDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)


#### `OpensDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)


#### `UsesDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)


#### `ProvidesDirectiveTree`

父类：[`ModuleDirectiveTree`](#moduledirectivetree)


#### 编译单元树`CompilationUnitTree`

父类：[`Tree`](#树tree)


#### 导入项树`ImportClauseTree`

父类：[`Tree`](#树tree)


#### 导入树`ImportTree`

父类：[`ImportClauseTree`](#导入项树importclausetree)


#### 模块声明树`ModuleDeclarationTree`

父类：[`Tree`](#树tree)


#### 包声明树`PackageDeclarationTree`

父类：[`Tree`](#树tree)


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


#### Enum常量树`EnumConstantTree`

父类：[`Tree`](#树tree)


#### 数组长度树`ArrayDimensionTree`

父类：[`Tree`](#树tree)



### 类型`Type`

父类：无

类型是把Java中的类型转换为语法树元素得来的，内置了多种判断类型的方法。

注意此处的类型`Type`和[`Tree.Kind`](#树tree)中的种类不是同一个东西：此处的类型是Java中的类型，[`Tree.Kind`](#树tree)中的种类是语法树中的种类。

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

符号表示一个方法、类、变量等等的签名信息。

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
    // 我们知道这个语法树的类型，所以可以强制转换；如果有多个类型，可以用instanceof判断
    MethodTree methodTree = (MethodTree) tree;
    // 做一系列的检查，如果检查到问题，就调用reportIssue方法
    if (methodTree.parameters().size() == 1) {
      Symbol.MethodSymbol methodSymbol = methodTree.symbol();
      Type firstParamType = methodSymbol.parameterTypes().get(0);
      Type returnType = methodSymbol.returnType().type();
      if (returnType.is(firstParamType.fullyQualifiedName())) {
        this.reportIssue(methodTree.simpleName(), "Parameter type must be different from return type");
      }
    }
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




## SonarQube自定义规则范例

所有规则均来源于[Java开发手册](https://github.com/alibaba/p3c)。

### 编程规约

#### 1.1.1 【强制】代码中的命名均不能以下划线或美元符号开始，也不能以下划线或美元符号结束。

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


#### 1.1.6 【强制】常量命名应该全部大写，单词间用下划线隔开，力求语义表达完整清楚，不要嫌名字长。

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


#### 1.1.11 【强制】避免在子父类的成员变量之间、或者不同代码块的局部变量之间采用完全相同的命名，使可理解性降低。

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


#### 1.2.2 【强制】long 或 Long 赋值时，数值后使用大写 L，不能是小写 l，小写容易跟数字混淆，造成误解。


#### 1.3.2 【强制】左小括号和右边相邻字符之间不需要空格；右小括号和左边相邻字符之间也不需要空格；而左大括号前需要加空格。

