---
layout: post
title: Lua元表与元方法
date: 2013-12-13 00:21
category: "lua"
---

<h2 id="tagline">About Lua Metatable</h2>
通常，Lua中的每个值都有一套预定义的操作集合。例如将数字相加，可以连接字符串，还可以在table中插入一对key-value等。但是我们无法将两个table相加，无法对
函数作比较，也无法调用一个字符串。__可以通过元表来修改一个值的行为，使其面对一个预定义的操作执行一个制定的操作。__例如，当Lua试图将两个table相加时，它
会先检查两者之一是否有元表，然后检查该元表中是否有一个叫__add的字段。如果找到了该字段，就调用该字段对应的值。这个值就是所谓的"元方法"，它应该是一个函数
，在本例中，这个函数用于计算table的和。

可以使用getmetatable获取一个值的元表，也可以使用setmetatable来设置或修改任何table的元表。

		t= {}
		printf(getmetatable(t))  --->nil 创建新的table时不会创建元表
		t1 = {}
		setmetatable(t,t1)  	--->设置t的元表为table t1
		assert(getmetatable(t)==t1) --->当t的元表不为t1时将触发错误

任何table可以作为任何值的元表而一组相关的table也可以共享一个通用的元表，元表描述它们的共同行为，一个table甚至可以作为它自己的元表，用于表述其特有的行为。

##算数类的元方法
假设用table来表示集合。并且有一些函数用来计算集合的并集和交集等。

		local mt = {}	--集合的元表
		Set = {} --创建名字空间

		--根据参数列表中的值创建一个新的集合
		function Set.new(l)
			local set = {}
			setmetatable(set,mt)
			for _,v in ipairs(l) do set[v] = true end
			return set
		end


		function Set.union(a,b)
			local res = Set.new{} 				
			for k in pairs(a) do res[k] = true end	--如果table a中key不为nil，则res[k]=true 
			for k in pairs(b) do res[k] = true end
			return res --返回a,b的并集
		end

		mt.__add = Set.union 	--只要Lua试图将两个集合相加，就会调用Set.union函数,使用加号(+)计算两个集合的交集，
								--需要让所有用于表示集合的table共享一个元表，并且在该元表中定义如何执行一个加法操作

		function Set.intersection(a,b)	
			local res = Set.new{}
			for k in pairs(a) do
			res[k] = b[k] --如果b中未定义k的值，则res[k]=nil
			end
			return res --返回a,b的交集
		end
		
		mt.__mul = Set.intersection --乘号来求集合的交集

		function Set.tostring(set)
			local l = {}
			for e in pairs(set) do
				l[#l+1] = e
			end
			return "{"..table.concat(1,", ").."}" --concat列出参数中指定table的数组部分从start位置到end位置的所有元素，元素间以, 隔开
		end

		function Set.print (s)
			print(Set.tostring(s))
		end

		s1 = Set.new{10,20,30,50} 	--返回table s1中 s1[10]=true s2[20]=true...
		s2 = Set.new{30,1}
		print(getmetatable(s1))		--打印字符串一样，说明创建的集合属于相同的元表
		print(getmetatable(s2))

		s3 = s1+s2
		Set.print(s3)	--->{1,10,20,30,50}	
		Set.print((s1+s2)*s3)	--->{10, 20, 30, 50} --注意打印的格式

* __add:加
* __mul:乘
* __sub:减
* __unm:相反数
* __div:除
* __mod:取模
* __pow:乘幂
* __concat:表述连接操作符的行为

##关系类的元方法
元表还可以表示关系操作符的含义，元方法为*__eq*(等于)，*__lt*(小于)，*__le*（小于等于)。与算数类的元方法不同的是，关系类的元方法不能应用于混合的类型。对于混合类型而言，关系类元方法的行为就模拟这些操作符在Lua中普通的行为。

###库定义的元方法
元表也是一种常规的table，所以任何人，任何函数都可以使用它们。

函数tostring示例:tostring能将各种类型的值表示为一种简单的文本格式，函数print总是调用tostring来格式化其输出。当格式化任意值时，tostring会检查该值是否有一个*__tostring*的元方法，如果有这个元方法，tostring就用该值作为参数来调用这个元方法。

        mt.__tostring = Set.tostring
        s1 = Set.new{10,4,5}
        print(s1)   --->{4,5,10}       --print调用tostring函数，进而调用Set.string
        
函数setmetatabe和getmetatabe会用到元表中的一个字段，用于保护元表。假设想要保护集合的元表，使用户既不能看也不能修改集合的元表。那么就需要用到字段__metatable。当设置该字段时，getmetatable就会返回这个字段的值，而setmetatable则会引发一个错误：

        mt.__metatable = "not your business"
        s1 = Set.new{}
        print(getmetatable(s1)) --->not your business 想要获得集合的元表
        setmetatable(s1,{})
        stdin:1: cannot change protected metatable 引发一个错误
     
###table访问的元方法
Lua提供了两种可以改变的table行为：查询table及修改table中不存在的字段

####__index元方法
当访问一个table中不存在的字段时，它会促使解释器去查找一个叫*__index*的元方法。如果没有这个元方法，那么访问结果为nil，否则，就有这个元方法来提供最终结果

假设要创建一些表述窗口的table，每个table中必须表述窗口参数，例如位置、大小及主题颜色等。所有这些参数都有默认值，因此希望在创建窗口对象时可以制定那些不同于默认值的参数。

        使用构造式
        Window = {} --创建一个名字空间
        --使用默认值创建一个原型
        Window.prototype = {x=0, y=0, width=100, height=100 }
        Window.mt = {}  --创建元表
        --声明构造函数
        function Window.new(o)
            setmetatable(o,Window.mt)
            return 0
        end
        --定义__index元方法
        Window.mt.__index = function (table,key)
            return Window.prototype[key]
        end
        
        w=Window.new{x=10,y=20}
        print(w.width)  --->100 访问w中未定义的width，返回元操作__index的返回值
        
当*__index*元方法为一个函数时，Lua以table和不存在的key作为参数调用这个函数，而当它是一个table时，Lua就以相同的方式来重新访问这个table，因此，上例中*__index*的声明可以简单地写为:*Window.mt.__index = Window.prototype*

如果不想在访问一个table时涉及到它的\__index元方法，可以使用函数rawget。调用rawget(t,i)就是对table t进行一个原始的（raw）访问。

####__newindex元方法
*__newindex*用于table的更新，*__index*用于table的查询，当对一个table中不存在的索引赋值时，解释器就会查找*__newindex*元方法。如果有这个元方法，解释器就调用它，而不是执行赋值。吐过这个元方法是一个table，解释器就会在此table中执行赋值，而不是对原来的table。

调用raw(t,k,v)就可以不设计任何元方法而直接设置table t中与key k想关联的value v.

组合使用*__index*和*__newindex*元方法可以实现出Lua中的一些强大功能，例如，只读的table、具有默认值的table和面向对象编程中的继承。
        
####具有默认值的table
常规table中的任何字段默认都是nil，通过元表就可以很容易的修改这个默认值：

        function setDefault (t,d)
            local mt={__index = function () return d end} --定义__inde元方法
            setmetatable(t,mt)
        end
        
        tab = {x=10,y=20}
        print(tab.x,tab.z)  --->10 nil
        setDefault(tab,0)   
        print(tab.x,tab.z)  --->10 0
        --调用setDefault后，任何对tab中不存在字段的访问都将调用它的__index元方法
        
setDefault函数为所有需要默认值的table创建了一个新的元表，如果准备创建很多需要默认值的table，这种方法的开销或许就比较大了。由于元表中默认值d是与元方法关联在一起的，所以setDefault无法为所有table都使用同一个元表。如果让具有不同默认值的table都使用同一个元表，那么就需要将每个元表的默认值都存放在table本身中。可以使用一个额外的字段来保存默认值。

        --如果不担心名字冲突的话，可以使用"___"这样的key作为额外的字段：
        local mt = {__index = function (t) return t.___ end}
            t.___ = d
            setmetatable(t,mt)
        end
        
        --如果担心名字冲突，那么报确保一个特殊key的唯一性也很容易，
        --只需创建一个新的table，并用它作为key即可
        local key = {}  --唯一的key
        local mt = {__index = function(t) return t[k] end}
        function setDefault (t,d)
            t[k] = d
            setmetatablt(t,mt)
        end

####跟踪table的访问
*__index*和*__newindex*都是在table中没有所需访问的index时才发挥作用的。因此，只有将一个table保持为空，才有可能捕捉到所有对它的访问。为了监视一个table的所有访问，就应该为真正的table创建一个代理。这个代理就是以个空的table，其中*__index*和*__newindex*元方法可用于跟踪所有的访问，并将访问重定向到原来的table上。

        t={}    --原来的table
        
        local _t =t     --保持对原table的一个私有访问
        t = {}  --创建代理
        
        --创建代理
        local mt = {
            __index = function(t,k)
            print("*access to element " .. tostring(k))
            return -t[k]    --访问原来的table
            end,
        
        __newindex = function(t,k,v)
            print("*update of element " .. tostring(k) .. " to " .. tostring(v)
        _t[k] = v   --更新原来的table
        end
        }
        setmetatablt(t,mt)
        
        t[2] = "hello"
        --->*update of element 2 to hello
        print(t[2])
        *access to element 2
        hello
        
上例方法存在一个问题，就是无法遍历原来的table。函数pairs只能操作代理table,而无法访问原来的table.

如果想要同时监视几个table，无需为每个table创建不同的元表。相反，只要以某种形式将每个代理与其原table关联起来，并且所有代理都共享一个公共的元表。

        local index = {}    --创建私有索引
        
        local mt ={         --创建元表
            __index = function (t,k)    
            print("*access to element " .. tostring(k))
            return t[index][k]  -- 访问原来的table
            end,
            
            __newindex = function(t,k,v)
                print("*update to element " .. tostring(k) .. " to " .. tostring(v)
            t[index][k] = v --更新原来的table
            end
       }
       
        function track(t)
            local proxy = {}
            proxy[index] = t
            setmetatablt(proxy,mt)
            return proxy
        end
        现在，若要监视table t，唯一要做的就是执行t = track(t)
        
####只读的table
实现只读的table,只要跟踪所有对table的更新操作，并引发一个错误就可以了。由于无需跟踪查询访问，所以对于*__index*元方法可以直接使用原table来代理函数。这种做法要求每个只读代理创建一个新的元表，其中*__index*指向原来的table

        function readOnly (t)
            local proxy = {}
            local mt = {    --创建元表
                __index = t
                __newindex = function(t,k,v)
                    error("attempt to update a read-only table",2)
                    end
            }
            setmetatable(proxy,mt)
            return proxy
        end
        
        --示例
        days = readOnly{"Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"}
        --这里days为一个代理
        print(day[1])   --->Sunday
        days[2] = "Noday"
        stdin:1:attempt to update a read-only table
        