## 两个对象的 hashCode()相同，则 equals()也一定为 true，对吗？

	答：不一定相同。正常情况下，因为equals()方法比较的就是对象在内存中的值，
		如果值相同，那么Hashcode值也应该相同。
		但是如果不重写hashcode方法，就会出现不相等的情况。
	
	在散列表中，hashCode()相等即两个键值对的哈希值相等，然而哈希值相等，并不一定能得出键值对相等。


### 
	当equals方法被重写时，通常有必要重写 hashCode 方法，
	以维护 hashCode 方法的常规协定，该协定声明相等对象必须具有相等的哈希码。 

	hashCode()方法就是为哈希表服务的．
	当我们在使用形如HashMap, HashSet这样前面以Hash开头的集合类时，
	hashCode()就会被隐式调用以来创建哈希映射关系．
	稍后我们再对此进行说明．这里我们先重点关注一下hashCode()方法的写法．
###

	＜Effective Java＞中给出了一个能最大程度上避免哈希冲突的写法，
	但我个人认为对于一般的应用来说没有必要搞的这么麻烦．如果你的应用中HashSet中
	需要存放上万上百万个对象时，那你应该严格遵循书中给定的方法．
	如果是写一个中小型的应用，那么下面的原则就已经足够使用了：
	
	要保证Coder对象中所有的成员都能在hashCode中得到体现．
	对于本例，我们可以这么写：
	@Override
	public int hashCode() {
		int result = 17;
		result = result * 31 + name.hashCode();
		result = result * 31 + age;
		
		return result;
	}

### 重写equals()而不重写hashCode()的风险
	在Oracle的Hash Table实现中引用了bucket的概念．

	在每个bucket上会挂一个链表，链表的每个结点都用来存放对象．
	Java通过hashCode()方法来确定某个对象应该位于哪个bucket中，
	然后在相应的链表中进行查找．在理想情况下，如果你的hashCode()方法写的足够健壮，
	那么每个bucket将会只有一个结点，这样就实现了查找操作的常量级的时间复杂度．
	即无论你的对象放在哪片内存中，我都可以通过hashCode()立刻定位到该区域，
	而不需要从头到尾进行遍历查找．这也是哈希表的最主要的应用．

	如果hashCode()一样的话，HashMap, HashSet等集合类就失去了其 "哈希的意义"．
	用<Effective Java>中的话来说就是，哈希表退化成了链表．
	如果hashCode()每次都返回相同的数，那么所有的对象都会被放到同一个bucket中，
	每次执行查找操作都会遍历链表，这样就完全失去了哈希的作用．


