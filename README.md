# SonarQube自定义规则简便文档

Sonar通过遍历语法树的形式来进行代码检查。

官方的sonar-java插件里已经内置了很多规则，可以在[这里](https://github.com/SonarSource/sonar-java/tree/master/java-checks/src/main/java/org/sonar/java/checks)看到源码。




## 语法树组件

### 树`Tree`

整个文件的语法树中的一个节点，表示一个语句、一个表达式、一个变量、一个方法、一个类等等。树的类型非常多，具体可以在`org.sonar.plugins.java.api.tree.BaseTreeVisitor`中找到。

所有的树类型的组件都可以使用`Tree`类的方法。

`Tree`下有`Tree.Kind`枚举类，包含了所有树节点的种类。注：后面会遇到[类型`Type`](#类型type)，和`Tree.Kind`不是同一个东西。为做区分，将`Tree.Kind`称为种类，将[`Type`](#类型type)称为类型。

* `boolean is(Kind... var1)`：判断这个节点的种类，比如`Tree.Kind.METHOD`、`Tree.Kind.CLASS`等；一次可传入多个类型，如果有一个种类匹配则返回`true`。

* `@Nullable Tree parent()`：获取父节点。

* `@Nullable SyntaxToken firstToken()`：获取这个节点的第一个[`SyntaxToken`](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken lastToken()`：获取这个节点的最后一个[`SyntaxToken`](#语法tokensyntaxtoken)。

* `Tree.Kind kind()`：获取这个节点的种类。


#### 方法树`MethodTree`

* `ModifiersTree modifiers()`：获取方法的[修饰符树](#修饰符)。

* `TypeParameters typeParameters()`：获取这个方法的泛型参数。

  `TypeParameters`是一个[`ListTree`](#列表树listtree)的子类，它的`openBracketToken()`和`closeBracketToken()`可以获取泛型参数的`<`和`>`的[语法token](#语法tokensyntaxtoken)，同时它可以使用所有[`ListTree`](#列表树listtree)的方法。

* `@Nullable TypeTree returnType()`：获取这个方法的返回值的[类型树](#类型树typetree)。

* `IdentifierTree simpleName()`：获取方法名的[标识符树](#标识符树identifiertree)。

* `List<VariableTree> parameters()`：获取这个方法的参数列表，以一列[变量树](#变量树variabletree)的方式列出。

* `SyntaxToken throwsToken()`：获取这个方法的`throws`关键字的[语法token](#语法tokensyntaxtoken)。

* `ListTree<TypeTree> throwsClauses()`：获取`throws`后面跟着的异常的[类型树](#类型树typetree)，以[列表树](#列表树listtree)的方式列出。

* `@Nullable BlockTree block()`：以[块树](#块树blocktree)的方式获取这个方法的内容。

* `@Nullable SyntaxToken defaultToken()`：获取这个方法的`default`关键字的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken openParenToken()`：获取这个方法的左括号的[语法token](#语法tokensyntaxtoken)。

* `@Nullable SyntaxToken closeParenToken()`：获取这个方法的右括号的[语法token](#语法tokensyntaxtoken)。

* `Symbol.MethodSymbol symbol()`：获取这个方法的[符号](#符号symbol)。


#### 类树`ClassTree`

* `ClassTree.simpleName()`：获取类名，类型是`IdentifierTree`

* `ClassTree.symbol()`：获取类的签名，含有类名、父类、接口、成员签名等信息

* `ClassTree.members()`：获取类的成员列表，类型是`List<Tree>` 

* `ClassTree.modifiers()`：获取类的修饰符，类型是`ModifierKeywordTree`

* `ClassTree.superClass()`：获取类的父类的类型，类型是`TypeTree`

* `ClassTree.superInterfaces()`：获取类的接口列表，类型是`List<TypeTree>`

* `ClassTree.typeParameters()`：获取类的泛型参数列表，类型是`List<TypeParameterTree>`


#### 变量树`VariableTree`

* `VariableTree.modifiers()`：获取变量的修饰符，类型是`ModifierKeywordTree`；`ModifierKeywordTree.keyword()`可以获取修饰符的名称

* `VariableTree.type()`：获取变量的类型，类型是`TypeTree`；`TypeTree.symbolType()`可以转换为`Type`类型

* `VariableTree.simpleName()`：获取变量名，类型是`IdentifierTree`

* `VariableTree.symbol()`：获取变量的符号，类型是`Symbol.VariableSymbol`；`VariableTree.symbol().type()`和`VariableTree.type().symbolType()`是等价的


#### 表达式树`ExpressionTree`

`ExpressionTree`有一系列不同的实现类，比如`LiteralTree`、`MemberSelectExpressionTree`、`NewClassTree`、`MethodInvocationTree`等等

通常可以通过`ExpressionTree.kind()`判断表达式树的精确种类，然后强制转换为对应的种类；或者通过`ExpressionTree.firstToken()`和`ExpressionTree.lastToken()`取得表达式部分token，进一步获得token的位置

* `ExpressionTree.symbolType()`：表达式的值的类型

* `ExpressionTree.asConstant()`：获取表达式的值

以下列出一些常用的子类：

##### `UnaryExpressionTree`

`operatorToken()`、`expression()`

##### `BinaryExpressionTree`

`operatorToken()`、`leftOperand()`、`rightOperand()`

##### `ConditionalExpressionTree`

##### `ArrayAccessExpressionTree`

##### `MemberSelectExpressionTree`


#### 语句树`StatementTree`

`StatementTree`有一系列不同的实现类，比如`AssertStatementTree`、`ReturnStatementTree`、`IfStatementTree`，`ForEachStatementTree`等等

`StatementTree`本身没有`Tree`之外的方法，但它的子类有很多方法，比如`IfStatementTree.thenStatement()`、`IfStatementTree.elseStatement()`、`IfStatementTree.condition()`等等

以下列出一些常用的子类：

##### `IfStatementTree`

`ifKeyword()`、`openParenToken()`、`closeParenToken()`、`thenStatement()`、`elseKeyword()`、`elseStatement()`、`condition()`

##### `AssertStatementTree`

`assertKeyword()`、`condition()`、`colonToken()`、`detail()`

##### `SwitchStatementTree`

`switchKeyword()`、`openParenToken()`、`expression()`、`closeParenToken()`、`openBraceToken()`、`cases()`、`closeBraceToken()`

本身没有方法，所用的方法全部来自于`SwitchTree`

##### `WhileStatementTree`

`whileKeyword()`、`openParenToken()`、`condition()`、`closeParenToken()`、`statement()`

##### `DoWhileStatementTree`

`doKeyword()`、`statement()`、`whileKeyword()`、`openParenToken()`、`condition()`、`closeParenToken()`

##### `ForStatementTree`

`forKeyword()`、`openParenToken()`、`variable()`、`expression()`、`colonToken()`、`closeParenToken()`、`statement()`

##### `BreakStatementTree`

`breakKeyword()`

##### `YieldStatementTree`

`yieldKeyword()`、`expression()`

##### `ContinueStatementTree`

`continueKeyword()`

##### `ReturnStatementTree`

`returnKeyword()`、`expression()`

##### `ThrowStatementTree`

`throwKeyword()`、`expression()`

##### `SynchronizedStatementTree`

`synchronizedKeyword()`、`openParenToken()`、`expression()`、`closeParenToken()`、`block()`

##### `TryStatementTree`

`tryKeyword()`、`openParenToken()`、`closeParenToken()`、`resourceList()`、`block()`、`catches()`、`finallyKeyword()`、`finallyBlock()`

#### 块树`BlockTree`

`BlockTree`是`StatementTree`的子类，有用的方法仅限于`BlockTree.body()`，使用它的树类很少

* `BlockTree.body()`：获取块的内容，类型是`List<StatementTree>`


##### 常量树`LiteralTree`

* `LiteralTree.value()`：获取常量的值，以字符串表示

* `LiteralTree.token()`：获取常量的token，类型是`SyntaxToken`；`SyntaxToken.text()`可以获取token的文本，等价于`LiteralTree.value()`


#### 标识符树`IdentifierTree`

* `IdentifierTree.name()`：标识符的名称，类型是`String`

* `IdentifierTree.symbol()`：标识符的符号


##### 调用方法树`MethodInvocationTree`

// TODO


#### 注解树`AnnotationTree`

* `TypeTree annotationType()`：获取注解的类型；`TypeTree.symbolType()`可以转换为`Type`类型

* `ListTree<ExpressionTree> arguments()`：获取注解的参数；`ListTree<ExpressionTree>`中包含一列`ExpressionTree`形式的参数


#### 修饰符树`ModifiersTree`

父类：[`ListTree`](#列表树listtree)

表示一系列的修饰符。

`ModifiersTree`是`ListTree`的子类，可以直接当作一个包含`ModifierTree`的`List`来使用。

* `List<AnnotationTree> annotations()`：获取修饰符中的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。

* `List<ModifierKeywordTree> modifiers()`：获取列表中的修饰符，修饰符以一列[修饰符关键字树](#修饰符关键字树modifierkeywordtree)的方式列出。

  用这个方法和直接把`ModifiersTree`当作`List`使用的区别是`List`中元素的类型。这个方法返回的是[`ModifierKeywordTree`](#修饰符关键字树modifierkeywordtree)，而直接当作`List`返回的则是`ModifierTree`。


#### 修饰符关键字树`ModifierKeywordTree`

父类：`ModifierTree`（`ModifierTree`相比它的父类[`Tree`](#树tree)没有任何新加的方法）

* `Modifier modifier()`：获取修饰符的类型。

  `Modifier`是一个枚举类，包含了所有修饰符的类型。

* `SyntaxToken keyword()`：获取修饰符的[语法token](#语法tokensyntaxtoken)。


#### 列表树`ListTree`

父类：[`Tree`](#树tree)、`List`

`ListTree`既能当作[`Tree`](#树tree)使用，也能当作`List`使用。

* `List<SyntaxToken> separators()`：获取列表中的分隔符，分隔符以一列[语法token](#语法tokensyntaxtoken)的方式列出。

  例如`ListTree`中有`a`、`b`、`c`三个元素，那么`separators()`返回的就是`a`和`b`之间的分隔符以及`b`和`c`之间的分隔符。


#### 类型树`TypeTree`

父类：[`Tree`](#树tree)

把节点的类型用树的方式存储。相比起[类型](#类型type)，类型树包含了更丰富的信息。

* `Type symbolType()`：获取类型树代表的[类型](#类型type)。

* `List<AnnotationTree>`：获取类型树的注解，注解以一列[注解树](#注解树annotationtree)的方式列出。



### 类型`Type`

父类：无

类型是把Java中的类型转换为语法树元素得来的，内置了多种判断类型的方法，常用于判断节点的类型；除此之外用处并不大。

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

  通过[`IdentifierTree`](#标识符树IdentifierTree)的`identifierToken()`可以获取符号每处的[语法token](#语法tokensyntaxtoken)，进而获得token的位置。

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

表示符号的元数据，用于获取符号的注解信息。

`SymbolMetadata`中定义了两个接口，`AnnotationInstance`和`AnnotationValue`，分别用于获取单个注解和单个注解的值。

* `boolean isAnnotatedWith(String annotationName)`：判断符号是否被`annotationName`注解了。

* `@CheckForNull List<AnnotationValue> valuesForAnnotation(String annotationName)`：获取符号被`annotationName`注解的值。

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

```
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

```
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

```
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

```
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

```
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

```
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

```
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

```
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

* [`ModifiersTree`](#修饰符树modifierstree)：`modifiers()`

* [`ModifierKeywordTree`](#修饰符关键字树modifierkeywordtree)：`modifier()`

```
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

```
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

