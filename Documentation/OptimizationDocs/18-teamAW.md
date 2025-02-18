# Структура для представления трехадресного кода и его генерация

## Постановка задачи

Реализация классов для представления и хранения строк трехадрессного кода, реализация визитора  для генерации строк трехадресного кода по AST.

## Команда — исполнитель

AW

## Зависимости

Зависит от:

- Парсер языка и генерация AST (M&M)  

Препятствует:

- Трехадресный код &mdash; это "язык оптимизаций", на базе него построена вся дальнейшая работа.

## Теория

Трехадресный код &mdash; это представление программы как последовательности операторов вида: `a = b op c`, где `a`, `b`, `c` &mdash; это имена переменных или константы, а `op` &mdash; это бинарная арифметическая или логическая операция.

Основным принципом трехадресного кода является то, что каждая строка не может содержать в себе более трех адресов.

Составные выражения при генерации трехадресного кода должны быть разбиты на подвыражения, каждое из которых должно быть присвоено соответствующей временной переменной, название у которых было принято единым: `tn`, где `n` &mdash; это уникальное для текущей временной переменной число-номер, обозначающий ее порядок при проведении генерации (нумерация временных переменных сквозная в рамках одного времени работы компилятора).

Помимо операторов присваивания с двумя операндами и бинарной операцией между ними, в трехадресном коде могут содержаться следующие конструкции:

- Метка строки `Ln:` для совершения перехода (каждая строка трехадресного кода может иметь метку);

- оператор безусловного перехода `goto Ln`;

- оператор условного перехода `if cond goto Ln`, где `cond` &mdash; это условие совершения прыжка к строке с меткой `Ln`;

- присваивания с унарными операциями `t1 = -t2`.

- вызовы функций в качестве первого и второго операндов оператора присваивания `a = func()`

## Реализация

Представление конструкций трехадресного кода (далее &mdash; ТАК) решено было выполнить в качестве объектов-наследников абстрактного класса:

```csharp
namespace SimpleLang.TACode.TacNodes
{
    /// <summary>
    /// Абстрактный класс для строки трехадресного кода
    /// </summary>
    public abstract class TacNode
    {
        /// <summary>
        /// Метка текущей строки трехадресного кода
        /// </summary>
        public string Label { get; set; } = null;
        /// <summary>
        /// Флаг-индикатор того, что данная метка была сгенерирована автоматически при обработке
        /// конструкций if/while/for
        /// </summary>
        public bool IsUtility { get; set; } = false;

        public override string ToString() => Label != null ? Label + ": " : "";
    }
}
```
Данный класс содержит лишь метку, как единственную общую характеристику каждой конструкции ТАК.
В качестве наследников `TacNode` были реализованы следующие классы:

- `TacAssignmentNode` &mdash; оператор присваивания

```csharp
namespace SimpleLang.TACode.TacNodes
{
    /// <inheritdoc />
    /// <summary>
    /// Представление оператора присваивания
    /// </summary>
    public class TacAssignmentNode : TacNode
    {
        /// <summary>
        /// Идентификатор адреса для записи вычисленного значения правой части
        /// </summary>
        public string LeftPartIdentifier { get; set; }
        /// <summary>
        /// первый операнд правой части
        /// </summary>
        public string FirstOperand { get; set; }
        /// <summary>
        /// Второй операнд правой части
        /// </summary>
        public string SecondOperand { get; set; } = null;
        /// <summary>
        /// Выполняемая операция
        /// </summary>
        public string Operation { get; set; } = null;

        public override string ToString()
        {
            var rightPart = (Operation == null) && (SecondOperand == null)
                ? $"{FirstOperand}"
                : $"{FirstOperand} {Operation} {SecondOperand}";
            return $"{base.ToString()}{LeftPartIdentifier} = {FirstOperand} {Operation} {SecondOperand}";
        }
    }
}
```

- `TacGotoNode` &mdash; оператор безусловного перехода

```csharp
namespace SimpleLang.TACode.TacNodes
{
    /// <inheritdoc />
    /// <summary>
    /// Представление оператора безусловного перехода goto
    /// </summary>
    public class TacGotoNode : TacNode
    {
        /// <summary>
        /// Метка для строки, на которую ведется переход
        /// </summary>
        public string TargetLabel { get; set; }
        
        public override string ToString() => $"{base.ToString()}goto {TargetLabel}";
    }
}
```

- `TacIfGotoNode` &mdash; оператор перехода по условию

```csharp
namespace SimpleLang.TACode.TacNodes
{
    /// <inheritdoc />
    /// <summary>
    /// Представление условного перехода if-goto
    /// </summary>
    public class TacIfGotoNode : TacGotoNode
    {
        /// <summary>
        /// Условие для перехода на целевую метку
        /// </summary>
        public string Condition { get; set; }

        public override string ToString() => (Label != null ? Label + ": " : "") + $"if {Condition} goto {TargetLabel}";
    }
}
```

- `TacEmptyNode` &mdash; пустой оператор, наследник `TacNode` в виде неабстрактного класса для возможности инстанцирования

В качестве контейнера для трехадресного кода используется класс `ThreeAddressCode`.

Он написан на основе связного списка `LinkedList` (get-only поле
`TACodeLines`) с элементами типа `<TacNode>` и реализует интерфейс
`IEnumerable`

Он предоставляет следующий функционал:

1. Доступ к элементам на чтение и модификацию;
2. модификация контейнера;
   1. добавление строки кода (нод) в конец списка `PushNode`;
   2. добавление нескольких строк кода (нод) в конец списка
      `PushNodes(IEnumerable)`;
   3. удаление ноды из списка `RemoveNode`;
   4. удаление ноды из списка по метке `RemoveNodeByLabel`;
   5. удаление нескольких нод из списка `RemoveNodes(IEnumerable)`;
3. набор методов для удобства добавления тривиальных нод;
4. поля First/Last для удобства получения соответствующих
   LinkedListNode<>.

```csharp
/// <summary>
/// Добавить несколько строк в конец контейнера
/// </summary>
/// <param name="nodes">Перечислимое строк, которые требуется добавить</param>
public void PushNodes(IEnumerable<TacNode> nodes)
{
    foreach (var tacNode in nodes)
    {
        TACodeLines.AddLast(tacNode);
    }
}

/// <summary>
/// Удаление строки по значению
/// <exception>InvalidOperationException</exception>
/// <exception>ArgumentNullException</exception>
/// </summary>
/// <param name="node">Строка, которую требуется удалить</param>
public void RemoveNode(TacNode node)
{
    TACodeLines.Remove(node);
}
```


Для генерации имен временных переменных и меток и обеспечения сквозной нумерации используется класс
`TmpNameManager`. 

Он реализует паттерн синглтон, доступ к методам происходит через
статическое поле `Instance`.

В нем заложен счетчик меток и переменных, для генерации нового имени
следует пользоваться методами.

- TmpNameManager.Instance.GenerateTmpVariableName()
- TmpNameManager.Instance.GenerateTmpLabel()

```csharp
namespace SimpleLang.TACode
{
    public class TmpNameManager
    {
        public static readonly TmpNameManager Instance = new TmpNameManager();

        /// <summary>
        /// Счетчики меток и переменных
        /// </summary>
        private int _currentVariableCounter = 0;
        private int _currentLabelCounter = 0;
        
        /// <summary>
        /// Генерация новой переменной с уникальным именем
        /// </summary>
        /// <returns>Уникальное имя временной переменной</returns>
        public string GenerateTmpVariableName() => $"t{++_currentVariableCounter}";
        
        /// <summary>
        /// Генерация новой метки
        /// </summary>
        /// <returns>Уничкальная метка</returns>
        public string GenerateLabel() => $"L{++_currentLabelCounter}";

        /// <summary>
        /// Сбросить счетчики
        /// </summary>
        public void Drop() { _currentLabelCounter = 0; _currentVariableCounter = 0; }
    }
}
```

Для генерации трехадресного кода используется визитор
`ThreeAddressCodeVisitor`. 

В нем переопределены соответствующие методы для обхода AST и генерации
TAC для 

- Операторов присваивания (с обработкой тривиальных кейсов и логических
  операций)
- Циклов for и while
- Уловного опретора if-else
- Оператора безусловного перехода goto;

А так же метод постообработки полученной программы для удаления лишних пустых операторов с метками, автоматическая генерация которых является недостатком рекурсивного подхода.
```csharp
public void Postprocess()
{
    var nodesToRemove = new List<TacNode>();
    // Обход всех строк сверху вниз
    var currentNode = TACodeContainer.First;
    while (currentNode != null)
    {
        var next = currentNode.Next;

        if (next == null)
        {
            currentNode = next;
            continue;
        }
        if (currentNode.Value is TacEmptyNode && !(next.Value is TacEmptyNode))
        {
            // Если встретили пустой оператор
            if (currentNode.Value.Label != null)
            {
                // Если у него есть метка -- "сливаем" его со следующим за ним оператором  
                next.Value.Label = currentNode.Value.Label;
                nodesToRemove.Add(currentNode.Value);
            }
        }
        currentNode = next;
    }
    // Удаляем все встреченные пустые операторы, так как их метки теперь указывают на строки кода 
    TACodeContainer.RemoveNodes(nodesToRemove);
}
```
Пример рекурсивной функции, генерирующей строки трехадресного кода по выражению.
```csharp
 /// <summary>
/// Генерация строк трехадресного кода по выражению
/// </summary>
/// <param name="expression">Целевое выражение, по которому будет проводится генерация</param>
/// <returns>Последний идентификатор, возвращенный процессом генерации</returns>
private string GenerateThreeAddressLine(ExprNode expression)
{
    string label = null;
    switch (expression)
    {
        // Тривиальные случаи
        case IdNode idNode:
        {
            return TACodeContainer.CreateAndPushIdNode(idNode, label);
        }
        case IntNumNode intNumNode:
        {
            return TACodeContainer.CreateAndPushIntNumNode(intNumNode, label);
        }
        case BoolNode boolNode:
        {
            return TACodeContainer.CreateAndPushBoolNode(boolNode, label);
        }
        // Унарный оператор
        case UnOpNode unOpNode:
        {
            var unaryExp = ManageTrivialCases(unOpNode.Unary);
            
            var tmpName = TmpNameManager.Instance.GenerateTmpVariableName();
            TACodeContainer.PushNode(new TacAssignmentNode()
            {
                Label = label,
                LeftPartIdentifier = tmpName,
                FirstOperand = null,
                Operation = unOpNode.Op,
                SecondOperand = unaryExp
            });
            return tmpName;
        }
        // Обработка сложного случая с бинарной операцией
        case BinOpNode binOpNode:
        {   
            var leftPart = ManageTrivialCases(binOpNode.Left);
            var rightPart = ManageTrivialCases(binOpNode.Right);
            
            // Создание и добавление в контейнер полученной бинарной операции между
            // двумя сгенерированными выше переменными
            var tmpName = TmpNameManager.Instance.GenerateTmpVariableName();
            TACodeContainer.PushNode(new TacAssignmentNode()
            {
                Label = label,
                LeftPartIdentifier = tmpName,
                FirstOperand = leftPart,
                Operation = binOpNode.Op,
                SecondOperand = rightPart
            });
            return tmpName;
        }
        // Обработка логического отрицания
        case LogicNotNode logicNotNode:
        {
            var unaryExp = ManageTrivialCases(logicNotNode.LogExpr);
            
            var tmpName = TmpNameManager.Instance.GenerateTmpVariableName();
            TACodeContainer.PushNode(new TacAssignmentNode()
            {
                Label = label,
                LeftPartIdentifier = tmpName,
                FirstOperand = null,
                Operation = "!",
                SecondOperand = unaryExp
            });
            return tmpName;
        }
    }

    return default(string);
}

```

## Тесты
### Input
```csharp
x = 42;
a = (x - 180) / 55 + x * 2;

if(x > 50){
    a = func2();
} else{
    a = 305;
}

b = false;
if(!b){
    b = b && (x == 0);
}

for(i = 0 to 100){
    x = i;
}    

goto l 120:

while(x < 50){
    x = x - 1;
    a = a + func();
}

l 120: x = 5;

```
### Output
```
x = 42
t1 = x - 180
t2 = t1 / 55
t3 = x * 2
t4 = t2 + t3
a = t4
t5 = x > 50
if t5 goto L1
a = 305
goto L2
L1: a = func2()
L2: b = False
t6 =  ! b
if t6 goto L3
goto L4
L3: t7 = x == 0
t8 = b && t7
b = t8
L4: i = 0
L5: x = i
i = i + 1
t9 = i < 100
if t9 goto L5
goto l120
L6: t10 = x < 50
if t10 goto L8
goto L7
L8: t11 = x - 1
x = t11
t12 = a + func()
a = t12
goto L6
L7:
l120: x = 5
```

## Вывод

Были реализованы структуры для предсавления и хранения программы в виде строк трехадресного кода. 

Был написан класс-визитор для обхода AST и генерации структур трехадресного кода.
