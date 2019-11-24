## We got screwed in the past

### case 1 - Integer

Everyting in python is object includes Integer, for comparing two Integers will need a operator '==' or 'is' and here is the difference:

- `is` is comparing two integer objs' id if they refer to same address in memory.
- `==` will check two integer objs' value if they are equal by calling obj's `__eq__()`.

so we should have enough knowledge to look at the code example below:

```Python
def main():
	x = y = -1
	while True:
		x += 1
		y += 1
		if x is y:
			print('%d is %d' % (x, y))
		else:
			print('Attention! %d is not %d' % (x, y))
			break
			
	x = y = 0
	while True:
		x -= 1
		y -= 1
		if x is y:
			print('%d is %d' % (x, y))
		else:
			print('Attention! %d is not %d' % (x, y))
			break


if __name__ == '__main__':
	main()

```

As the result shows below, `is` and `==` does not work. The reasons why we have such bug is because Python use a linked-list called `small_ints` to cache integer objs which are in frequent use during the whole lifecycle, whenever those integer objs need, there is no new obj will be created but refer to the Integer objs in `small_ints`, this is for high efficience data process but Python utilize a limited range [-5, 256] for Integer obj frequent use therefore even they have same value but different object.

![](./res/int-is-comparation.png)

of course this is just an appetizer, let's take a look at another example below

```Python
import dis
a = 257


def main():
	b = 257  # 6th line
	c = 257  # 7th line
	print(b is c)  # True
	print(a is b)  # False
	print(a is c)  # False


if __name__ == "__main__":
	main()

```

Even `a`,`b` and `c` have same value, only `b is c` return True others return False, why is that!? First of all we have to talk about the code block which is a piece of Python program text that is executed as a unit, due to the performance reasons Python create an Integer obj in a code block if the value is not in the range of [-5, 256] as in this code block if a same value Integer obj exist already then will let that new Integer obj refer to this one otherwise create a new one, and just note this rule does not apply to negative numbers. However, `b is c` return True but `a` and `b` doesn't because they are not in same code block.
For closer look, we could use `dis` module go through the process via checking the bytecode, if you don't touch bytecode before take look at Python Doc [Disassembler for Python bytecode]((https://docs.python.org/2/library/dis.html?highlight=bytecode))

```Python
if __name__ == "__main__":
	main()
	dis.dis(main)

```
![](./res/result-of-dis.png)

Here you could get more info related to Python integer implementation [Python integer objects implementation](https://www.laurentluce.com/posts/python-integer-objects-implementation)

### 坑2 - 嵌套列表的坑

Python中有一种内置的数据类型叫列表，它是一种容器，可以用来承载其他的对象（准确的说是其他对象的引用），列表中的对象可以称为列表的元素，很明显我们可以把列表作为列表中的元素，这就是所谓的嵌套列表。嵌套列表可以模拟出现实中的表格、矩阵、2D游戏的地图（如植物大战僵尸的花园）、棋盘（如国际象棋、黑白棋）等。但是在使用嵌套的列表时要小心，否则很可能遭遇非常尴尬的情况，下面是一个小例子。

```Python
def main():
	names = ['关羽', '张飞', '赵云', '马超', '黄忠']
	subjs = ['语文', '数学', '英语']
	scores = [[0] * 3] * 5
	for row, name in enumerate(names):
		print('请输入%s的成绩' % name)
		for col, subj in enumerate(subjs):
			scores[row][col] = float(input(subj + ': '))
	print(scores)


if __name__ == '__main__':
	main()

```

我们希望录入5个学生3门课程的成绩，于是定义了一个有5个元素的列表，而列表中的每个元素又是一个由3个元素构成的列表，这样一个列表的列表刚好跟一个表格是一致的，相当于有5行3列，接下来我们通过嵌套的for-in循环输入每个学生3门课程的成绩。程序执行完成后我们发现，每个学生3门课程的成绩是一模一样的，而且就是最后录入的那个学生的成绩。

要想把这个坑填平，我们首先要区分对象和对象的引用这两个概念，而要区分这两个概念，还得先说说内存中的栈和堆。我们经常会听人说起“堆栈”这个词，但实际上“堆”和“栈”是两个不同的概念。众所周知，一个程序运行时需要占用一些内存空间来存储数据和代码，那么这些内存从逻辑上又可以做进一步的划分。对底层语言（如C语言）有所了解的程序员大都知道，程序中可以使用的内存从逻辑上可以为五个部分，按照地址从高到低依次是：栈（stack）、堆（heap）、数据段（data segment）、只读数据段（static area）和代码段（code segment）。其中，栈用来存储局部、临时变量，以及函数调用时保存现场和恢复现场需要用到的数据，这部分内存在代码块开始执行时自动分配，代码块执行结束时自动释放，通常由编译器自动管理；堆的大小不固定，可以动态的分配和回收，因此如果程序中有大量的数据需要处理，这些数据通常都放在堆上，如果堆空间没有正确的被释放会引发内存泄露的问题，而像Python、Java等编程语言都使用了垃圾回收机制来实现自动化的内存管理（自动回收不再使用的堆空间）。所以下面的代码中，变量`a`并不是真正的对象，它是对象的引用，相当于记录了对象在堆空间的地址，通过这个地址我们可以访问到对应的对象；同理，变量`b`是列表容器的引用，它引用了堆空间上的列表容器，而列表容器中并没有保存真正的对象，它保存的也仅仅是对象的引用。

 ```Python
a = object()
b = ['apple', 'pitaya', 'grape']
 ```

知道了这一点，我们可以回过头看看刚才的程序，我们对列表进行`[[0] * 3] * 5`操作时，仅仅是将`[0, 0, 0]`这个列表的地址进行了复制，并没有创建新的列表对象，所以容器中虽然有5个元素，但是这5个元素引用了同一个列表对象，这一点可以通过`id`函数检查`scores[0]`和`scores[1]`的地址得到证实。所以正确的代码应该按照如下的方式进行修改。

```Python
def main():
	names = ['关羽', '张飞', '赵云', '马超', '黄忠']
	subjs = ['语文', '数学', '英语']
	scores = [[]] * 5
	for row, name in enumerate(names):
		print('请输入%s的成绩' % name)
		scores[row] = [0] * 3
		for col, subj in enumerate(subjs):
			scores[row][col] = float(input(subj + ': '))
	print(scores)


if __name__ == '__main__':
	main()

```

或者

```Python
def main():
	names = ['关羽', '张飞', '赵云', '马超', '黄忠']
	subjs = ['语文', '数学', '英语']
	scores = [[0] * 3 for _ in range(5)]
	for row, name in enumerate(names):
		print('请输入%s的成绩' % name)
		scores[row] = [0] * 3
		for col, subj in enumerate(subjs):
			scores[row][col] = float(input(subj + ': '))
	print(scores)


if __name__ == '__main__':
	main()

```

如果对内存的使用不是很理解，可以看看[PythonTutor网站](http://www.pythontutor.com/)上提供的代码可视化执行功能，通过可视化执行，我们可以看到内存是如何分配的，从而避免在使用嵌套列表或者复制对象时可能遇到的坑。

![](./res/python-tutor-visualize.png)

![](./res/python-tutor-visualize2.png)

### 坑3 - 访问修饰符的坑

用Python做过面向对象编程的人都知道，Python的类提供了两种访问控制权限，一种是公开，一种是私有（在属性或方法前加上双下划线）。而用惯了Java或C#这类编程语言的人都知道，类中的属性（数据抽象）通常都是私有的，其目的是为了将数据保护起来；而类中的方法（行为抽象）通常都是公开的，因为方法是对象向外界提供的服务。但是Python并没有从语法层面确保私有成员的私密性，因为它只是对类中所谓的私有成员进行了命名的变换，如果知道命名的规则照样可以直接访问私有成员，请看下面的代码。

```Python
class Student(object):

    def __init__(self, name, age):
        self.__name = name
        self.__age = age

    def __str__(self):
        return self.__name + ': ' + str(self.__age)


def main():
    stu = Student('骆昊', 38)
    # 'Student' object has no attribute '__name'
    # print(stu.__name)
    # 用下面的方式照样可以访问类中的私有成员
    print(stu._Student__name)
    print(stu._Student__age)


if __name__ == '__main__':
    main()

```

Python为什么要做出这样的设定呢？用一句广为流传的格言来解释这个问题：“We are all consenting adults here”（我们都是成年人）。这句话表达了很多Python程序员的一个共同观点，那就是开放比封闭要好，我们应该自己对自己的行为负责而不是从语言层面来限制对数据或方法的访问。

所以在Python中我们实在没有必要将类中的属性或方法用双下划线开头的命名处理成私有的成员，因为这并没有任何实际的意义。如果想对属性或方法进行保护，我们建议用单下划线开头的受保护成员，虽然它也不能真正保护这些属性或方法，但是它相当于给调用者一个暗示，让调用者知道这是不应该直接访问的属性或方法，而且这样做并不影响子类去继承这些东西。

需要提醒大家注意的是，Python类中的那些魔法方法，如`__str__`、`__repr__`等，这些方法并不是私有成员哦，虽然它们以双下划线开头，但是他们也是以双下划线结尾的，这种命名并不是私有成员的命名，这一点对初学者来说真的很坑。
