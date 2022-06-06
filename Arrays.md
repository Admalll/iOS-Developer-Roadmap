# Массивы
`Array` - структура, тип значения, является универсальным контейнером общего назначения для хранения **упорядоченной** коллекции элементов, 
где через него можно выполнить итерацию по крайней мере один раз.

![image](https://user-images.githubusercontent.com/47610132/168491765-bc2d24e5-6ba5-47b6-adae-f7cc1d6b0544.png)

  - `_ContiguousArrayStorage` - выделение памяти для хранения элементов, обеспечение быстрого доступа по индексу.
  - `_ArrayBridgeStorage` - абстракция, позволяющая использовать как нативное хранилище, так и **NSArray**.
  - `_ArrayBuffer<T>` - реализация copy on write.
  - `Array<T>` - публичный интерфейс массива.

# Размерность массива
Каждый массив резервирует определенный объем памяти для хранения его содержимого. 
Когда вы добавляете элементы в массив и этот массив начинает превышать свою зарезервированную емкость, массив выделяет большую область памяти 
и копирует свои элементы в новое хранилище, где новое хранилище кратно размеру старого хранилища.

![image](https://user-images.githubusercontent.com/47610132/168482804-e961d8fe-bf1c-44d6-aca6-b4e37ea8a1a2.png)

Эта экспоненциальная стратегия роста означает, что добавление элемента происходит за постоянное время `O(1)`, усредняя производительность многих операций добавления. 
Операции добавления, запускающие перераспределение, имеют затраты на производительность, но они происходят все реже по мере увеличения массива.

Пример:
Создадим массив с десятью элементами. Swift выделит этому массиву достаточную емкость для хранения только этих десяти элементов,
таким образом и для `array.capacity` и для `array.count` будут равны 10.

```
var array = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

print(array.capacity) // 10
print(array.count) // 10
```
Давайте добавим **11** и **12** элементы. Наш массив не имеет для этого емкости, поэтому ему нужно освободить место - он найдет память для хранения большего 
количества элементов, скопирует массив туда, а затем добавит **11** и **12** элементы, где время выполнения этих вставок составляет сложность - `O(n)`, где n - количество элементов в массиве.

```
array.append(11)
array.append(12)

print(array.capacity) // 20
print(array.count) // 12
```

Таким образом, при добавлении 11 и 12 элементов в массив с **емкостью 10**, swift создаст массив с размером **20**. 
И когда мы превысим этот размер, то следующая ёмкость будет 40, потом 80, пото 160 и так далее.

> Если приблизительно известно, сколько элементов необходимо сохранить, используйте метод `reserveCapacity(_:)` перед добавлением в массив, 
чтобы избежать промежуточных перераспределений.
>> Используйте свойства `capacity` и `count`, чтобы определить, сколько элементов массив может хранить без выделения больших ресурсов.

```
var stringArray = Array<String>()
stringArray.reserveCapacity(128)
```

Для массивов с типом `T`, который является ссылочным типом, например `класс` или типом протокола `@objc`, 
это место хранения может быть смежным блоком памяти или экземпляром `NSArray`, где **NSArray** - это неизменяемый класс, ссылочного типа, а `NSMutableArray` - изменяемый сабкласс от NSArray. 
Поскольку любой произвольный подкласс **NSArray** может стать **Array**, в этом случае нет никаких гарантий относительно представления или эффективности в данном кейсе.

Примечание:
`Array<T>` подобен `ContiguousArray<T>`, если **T** не является **ссылочным типом** или **объектом Objective-C**.
В противном случае он может использовать для хранения «NSArray», подключенный от Cocoa.

![image](https://user-images.githubusercontent.com/47610132/168491933-cb4cfbfe-07fd-434d-a292-99fddba40418.png)

`ContiguousArray<T>` - является специализированным массивом, который всегда хранит свои блоки памяти непрерывно. Если тип элемента массива является ссылочным типом или протоколом @objc и нет необходимости соединять массив с `NSArray` или передавать массив к `Objective-C API`, то использование `ContiguousArray` может быть более эффективным и иметь более предсказуемую производительность, чем Array. Если тип элемент массива является тип значения, например **структура** или **перечисление**, то Array и ContiguousArray должны иметь одинаковую эффективность.

Список, когда `Array` хранится непрерывно, подобно `ContiguousArray` хранению:
  - Массив созданный в Swift
  - Элементами массива являются тип - значения: структуры, перечесления и etc.
  - Массивы на платформах без среды выполнения Objective-C

Единственный раз, когда `Array` не будет непрерывно храниться - это если его элемент ссылочного типа, например классы и был подключен к `NSArray`. Но даже тогда, во многих случаях, `NSArray` будет непрерывно храниться. Этот случай называю "ленивый мост" (lazy bridging), поэтому в свифте и есть надобность в `ContiguousArray`, т.к оптимизатор не может устранить ветвь, которая проверяет ленивый мост для каждого элемента, получаемого из массива. Например, при итерации, когда этот массив содержит классы, так что иногда можно обнаружить, что использование `ContiguousArray` дает небольшое ускорение для некоторого кода. Обычно это нано-оптимизация, а также никогда не применяется в Linux или с массивами структур.

Сравнение **Array** и **ContiguousArray**, так же **lazy Array** и **lazy ContiguousArray**:

```
import Foundation

protocol Possibles {
    init(repeating: Bool, count: Int)
    subscript(index: Int) -> Bool { get set }
    var count: Int { get }
}
extension Array: Possibles where Element == Bool {}
extension ContiguousArray: Possibles where Element == Bool {}

func lazySieveOfEratosthenes<Storage: Possibles>(makeStorage: () -> Storage) -> [Int] {
    var possibles = makeStorage()
    let limit = possibles.count - 1
    return (2 ... limit).lazy.filter { i in // Lazy, so that `possibles` is updated before it is used.
        possibles[i]
    }.map { i in
        stride(from: i * i, through: limit, by: i).lazy.forEach { j in
            possibles[j] = false
        }
        return i
    }.reduce(into: [Int]()) { (result, next) in
        result.append(next)
    }
}

func forSieveOfEratosthenes<Storage: Possibles>(makeStorage: () -> Storage) -> [Int] {
    var possibles = makeStorage()
        let limit = possibles.count - 1
        var result = [Int]()
        for i in 2 ... limit where possibles[i] {
            var j = i * i
            while j <= limit {
                possibles[j] = false
                j += i
            }
            result.append(i)
        }
        return result
}

func test(_ name: String, _ storageType: String, _ function: (Int) -> [Int]) {
    let start = Date()
    let result = function(100_000)
    let end = Date()
    print("\(name) \(storageType) biggest: \(result.last ?? 0) time: \(end.timeIntervalSince(start))")
}
test("for   ", "Array          ") { limit in
    forSieveOfEratosthenes {
        Array(repeating: true, count: limit + 1)
    }
}
test("for   ", "ContiguousArray") { limit in
    forSieveOfEratosthenes {
        ContiguousArray(repeating: true, count: limit + 1)
    }
}
test("lazy  ", "Array          ") { limit in
    lazySieveOfEratosthenes {
        Array(repeating: true, count: limit + 1)
    }
}
test("lazy  ", "ContiguousArray") { limit in
    lazySieveOfEratosthenes {
        ContiguousArray(repeating: true, count: limit + 1)
    }
}
```
Результат:
```
for    Array           biggest: 99991 time: 41.016937017440796
for    ContiguousArray biggest: 99991 time: 40.648478984832764
lazy   Array           biggest: 99991 time: 3.3549970388412476
lazy   ContiguousArray biggest: 99991 time: 3.5851539373397827
```
Еще один прогон и результат:
```
for    Array           biggest: 99991 time: 41.801795959472656
for    ContiguousArray biggest: 99991 time: 42.37710893154144
lazy   Array           biggest: 99991 time: 3.438219904899597
lazy   ContiguousArray biggest: 99991 time: 3.4085270166397095
```
Как видно, не всегда `Array` медленнее, чем `ContiguousArray`
> Прогон проходил на такой спеке:
  - macOS Monterey (Version 12.3)
  - Processor 2.3 GHz 8-Core Intel core i9
  - Memory 16 GB DDR4
  - Xcode Version 13.3.1 (13E500a), Playground

Пример кода взят [отсюда](https://forums.swift.org/t/execution-time-contiguousarray-vs-array/12467/21), советую ознакомиться со всем тредом)

P.S:
![image](https://user-images.githubusercontent.com/47610132/172070126-e958aae8-2600-4ad1-a1f5-b97f27f02ea2.png)
По-поводу того, что `ContiguousArray` оказался медленне `Array` я нашел только вот такое, небольшое сообщение, но не стоит брать эти слова за догмму, так как, возможно, код выше где-то оказался неточным 🤔

# Массив с классами

Если элементы в массиве являются **экземплярами класса**, семантика одинакова, хотя сначала они могут выглядеть разными. В этом случае значения, хранящиеся в массиве, являются ссылками на объекты, находящиеся вне массива. При изменении ссылки на объект в одном массиве **только этот массив** имеет ссылку на новый объект. Однако если два массива содержат ссылки на один и тот же объект, можно наблюдать изменения свойств этого объекта из обоих массивов.

```
class MyClass {
    var name = "Name"
}

var firstArray = [MyClass(), MyClass()]
var secondArray = firstArray

firstArray[0].name = "Another Name"
print(firstArray[0].name) // "Another name"
print(secondArray[0].name) // "Another name
```
  - Замены, добавления и удаления имеют область видимости только для массива, к которому применяется действи:
```
firstArray[0] = MyClass()

print(firstArray[0].name) // "Name"
print(secondArray[0].name) // "Another name
```

Операции с массивом и их сложность:
  - `O(1) — Константное время(самое быстрое)` Доступ к элементу в массиве по его индексу:
```
let array = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

array[3]
```
  - `O(n log n) - Логарифмическое время(немного по-хуже, чем константное время)` Сортировка по возрастанию, стандартная функция:
```
var people = ["Sandra", "Mike", "James", "Donald"]
people.sort()
```
  - `O(n) - Линейное время(немного по-хуже, чем логарифмическая сложность)` Подсчёт суммы элементов, при помощи стандартной функции `forEach` или же for i in array:
```
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
var sumOfNumbers = 0

numbers.forEach {
    sumOfNumbers += $0
}

print(sumOfNumbers) // 55
```
  - `O(n²) - Квадратичное время(немного по-хуже, чем O(n log n)` Прохождение по 2D массиву:
```
var twoDimensionalArray = [["first one", "second one"], ["first two", "second two"], ["first third", "second third"]]

for i in 0..<twoDimensionalArray.count {
    for n in 0..<twoDimensionalArray[i].count {
        print(twoDimensionalArray[i][n])
    }
}
```

# Заключение

Массивы в Swift являются хорошим типом данных для организации упорядоченной коллекции, где можно хранить самые разные элементы, но помним про семантику с `value` и `reference` типами, можно проводить самые разные операции ( добавление, удаление и т.д ), а так же использовать не только `Array`, но и `ContiguousArray`, `NSArray`, `ArraySlice` и т.д

После прочтения данного материала, я надеюсь, что ваши познания о массивах станут нмножечко глубже, вы сможете более качественно и продуманно писать код, а так же удивить интервьюера, когда тот спросит о массивах, благодарю вас за уделенное время и всего вам хорошего :)

# Дополнительные ресурсы:
 - [Массивы в Swift и операции с ними](https://medium.com/codex/swift-101-working-with-arrays-9586725e6eb0)
 - [Основы Swift - Массивы](https://www.youtube.com/watch?v=lscC8c3fNZU)
 - [Описание сложности алгоритмов в swift](https://github.com/raywenderlich/swift-algorithm-club/blob/master/Big-O%20Notation.markdown)
