# IndexedDB

## 打开数据库

1. 语法

   ```javascript
   let openRequest = indexedDB.open(databasename, ?version)
   ```

   第一个参数databasename表示要连接的数据库的名称，string类型的值。如果当前的源（域/协议/端口）下没有名称为databasename的数据库且连接请求成功，则会在当前的源下创建名称为databasename的数据库。

   第二个参数version表示要连接的数据库的版本，number类型的值。可选，不传则默认使用1。

   返回值为一个连接请求对象。

2. 连接请求对象的处理程序

   1. `onupgradeneeded`

      当传入的version参数大于要连接的数据库的本地版本时，openRequest的onupgradeneeded处理程序会被触发。

   2. `onsuccess`

      当连接请求成功时，openRequest的onsuccess处理程序会被触发。

   3. `onerror`

      当传入的version参数小于要连接的数据库的本地版本时，opneRequest的onerror处理程序会被触发。

   4. `onblocked`

      onblocked处理程序被触发的一个场景：某站点在浏览器中被打开且建立了一个和某数据库本地版本的连接，保持已有的标签页不关闭，然后更新站点的脚本，在更新后的脚本中会发起和该数据库更高版本的连接请求，之后在浏览器中新开一个标签页打开该站点，下载到更新后的脚本。此时，如果在旧标签页中下载的老脚本中没有针对已存在连接的数据库的版本变更的处理程序，则新的和更高版本的连接请求会被阻塞，触发onblocked处理程序。

      进一步的阐述和示例代码：

      对于一个数据库，在尝试打开和高于本地版本的版本的连接时，如果存在和本地版本的连接，则会触发所有和本地版本的连接的onversionchange处理程序，当所有和本地版本的连接都关闭时，打开和高于本地版本的版本的连接的请求的onupgradeneeded处理程序才会被触发，否则会触发该请求的onblocked处理程序。

      示例代码如下：

      ```javascript
      // 先执行以下代码
      // 假设某网站本地有foo数据库，且本地的版本为4
      let openRequest = indexedDB.open('foo', 4);
      let db, dbLeftToTest;
      openRequest.onsuccess = function() {
          db = openRequest.result;
          dbLeftToTest = openRequest.result;
      
          db.onversionchange = function() {
              db.close();
              alert('Database is outdated, please reload the page.')
          }
      }
      
      // 执行完上一部分代码之后，执行下列代码
      // 尝试打开和更高版本的foo数据库的连接
      // 由于上面的dbLeftToTest没有onversionchange处理程序，所以openRequest2请求会因为dbLeftToTest没有关闭而阻塞
      let openRequest2 = indexedDB.open('foo', 5);
      let db2;
      
      openRequest2.onupgradeneeded = function() {
          console.log('upgrade needed');
      
          db2 = openRequest2.result;
      }
      
      openRequest2.onerror = function() {
          console.log(openRequest2.error);
      }
      
      openRequest2.onsuccess = function() {
          console.log('success!');
      }
      
      openRequest2.onblocked = function() {
          console.log('blocked!');
      }
      ```

3. 删除数据库

   `let deleteRequest = indexedDB.deleteDatabase(databasename)`，可以在`deleteRequest.onsuccess`或`deleteRequest.onerror`中追踪结果。

## 对象库（object store）

1. 对象库（object store）是indexedDB的核心功能特性，相当于传统数据数据中的表或集合的概念。一个数据库当中可能有多个对象库。

2. 对象库可以存储几乎任何类型的值。IndexedDB使用标准序列化算法来克隆和存储对象，因此，存在循环引用的对象不能被存储。

3. 对象库中的每个值都需要有唯一的键，键字段的值需要是下列5种值之一：数字、字符串、Date对象、Array实例或者ArrayBuffer实例。

4. 创建对象库

   ```javascript
   let store = db.createObjectStore(name[, keyOption])
   ```

   - 第一个参数name表示对象库的名称。第二个参数keyOption是可选的，如果提供，则需要是一个包含keyPath属性或autoIncrement属性的对象。
   - 创建对象库、修改对象库（在对象库上创建索引等）和删除对象库的操作只能在连接请求的onupgradeneeded处理程序中进行。

5. 往对象库中添加值

   1. 对于一个对象库，往其中添加值的限制根据其被创建时提供的keyOption的不同，可以分3种情况：

      - 如果不提供keyOption参数，则在往该对象库添加值时需要显式地提供键。

      - 如果提供的keyOption是一个包含keyPath属性的对象，则往该对象库添加的值需要是一个存在keyPath所指示的键且键值唯一的对象。例如，如果将keyOption指定为`{keyPath: 'id'}`，则添加的对象需要有id属性且属性值唯一；如果将keyOption指定为`{keyPath: 'profile.uid'}`，则添加的对象中的profile属性需要为一个嵌套对象，嵌套对象内提供唯一的uid属性值。

      - 如果提供的keyOption是一个包含autoIncrement属性的对象，则可以往该对象库添加任意值。

   2. 往一个对象库中添加值常用有2个方法：

      - `store.add(value[, key])` 对`value`的限制以及是否需要传入`key`参照第1点
      - `store.put(value[, key])` 对`value`的限制以及是否需要传入`key`参照第1点

      add方法和put方法的区别：当使用add方法往对象库中添加一个关联的键已经存在于对象库中的值时，会抛出一个ConstraintError错误类实例；当使用put方法往对象库中添加一个关联的键已经存在于对象库中的值时，会将旧值覆盖。

   3. 往对象库中添加值的操作需要在事务内进行。

6. 创建对象库和往对象库中添加值的示例代码

   ```javascript
   let openRequest = indexedDB.open('foo', 9);
   let db;
   
   openRequest.onupgradeneeded = function() {
       db = openRequest.result;
       
       // avt: anyValueTestStore
       if(!db.objectStoreNames.contains('avt')) {
           db.createObjectStore('avt', {autoIncrement: true});
       }
       
       // ckpt: complicated key path test store
       if(!db.objectStoreNames.contains('ckpt')) {
           db.createObjectStore('ckpt', {keyPath: 'profile.uid'})
       }
   }
   
   openRequest.onsuccess = function() {
       console.log('open success');
       
       db = openRequest.result;
       
       let transaction = db.transaction(['avt', 'ckpt'], 'readwrite');
       let avtStore = transaction.objectStore('avt');
       let ckptStore = transaction.objectStore('ckpt');
   
       let req1 = avtStore.add('asfdfsdfd');
       req1.onsuccess = function() {
           console.log('req1\'s result: ', req1.result);
       }
   
       let req2 = avtStore.add({name: 'sfsd', age: 12});
       req2.onsuccess = function() {
           console.log('req2\'s result: ', req2.result);
       }
   
       let req3 = ckptStore.add({profile: {uid: '1'}, name: 'def'});
       req3.onsuccess = function() {
           console.log('req3\'s result: ', req3.result);
       }
       req3.onerror = function(e) {
           e.preventDefault();
           console.log('req3\'s error: ', req3.error);
       }
       
       let req4 = ckptStore.add({uid: '1', name:'abc'});
       req4.onerror = function(e) {
           e.preventDefault();
           console.log('req4\'s error: ', req4.error);
       }
   }
   ```



## 事务（transaction）

1. 事务是一组数据操作请求的组合。一般情况下，如果一个事务内的数据操作请求全部成功通过，则事务被提交并关闭，数据库状态更新；如果一个事务内的任意一个数据操作请求失败，则事务被取消并关闭，数据库状态回滚。

2. 在IndexedDB中，所有的数据及数据库操作请求必须在事务中进行。

3. `onupgradeneeded`处理程序被执行时，会自动创建versionchange事务。在versionchange事务内，可以做改变数据库的结构、创建/修改/删除对象库等操作。versionchange事务不能被手动创建。

4. 创建事务

   `let transaction = db.transaction(store/stores[, mode])`

   第一个参数可以传入一个对象库的名称，也可以传入一个由多个对象库的名称组成的数组。第二个参数表示创建的事务的模式，可以指定为'readonly'或者'readwrite'，若不指定则默认使用'readonly'。

   多个readonly类型的事务可以同时访问存储区，而一个readwrite类型的事务对存储区进行访问时，其他的事务需要等待。

5. 事务的相关操作与处理程序

   1. 监听事务的完成

      `transaction.oncomplete = () => {...}`

   2. 手动中止事务

      `transaction.abort()` 会撤销该事务内的所有操作

   3. 监听事务的中止

      `transaction.onabort = () => {...}` 在手动中止事务或者由于事务内的数据操作出错导致事务自动中止时被触发



## 事务的自动提交

1. 对于一个事务，当事务内的数据操作全部完成且微任务队列为空时，引擎会自动将该事务提交并关闭。

2. 当在数据库操作之间插入了网络请求、setTimeout等宏任务时，当宏任务开始执行时，事务已经被关闭。示例代码如下：

   ```javascript
   let openRequest = indexedDB.open('foo', 9);
   let db;
   
   openRequest.onupgradeneeded = function() {
       db = openRequest.result;
       
       // avt: anyValueTestStore
       if(!db.objectStoreNames.contains('avt')) {
           db.createObjectStore('avt', {autoIncrement: true});
       }
   }
   
   openRequest.onsuccess = async function() {
       console.log('open success');
       
       db = openRequest.result;
       
       let transaction = db.transaction('avt', 'readwrite');
       let avtStore = transaction.objectStore('avt');
       let ckptStore = transaction.objectStore('ckpt');
   
       let req1 = avtStore.add('asfdfsdfd');
       req1.onsuccess = function() {
           console.log('req1\'s result: ', req1.result);
       }
   
       // await new Promise(resolve => resolve(123)); // 这样不会导致事务的关闭，因为没有宏任务的执行
       await new Promise(resolve => {
           setTimeout(resolve, 1000);
       });
   
       let req2 = avtStore.add({name: 'sfsd', age: 12});
       req2.onsuccess = function() {
           console.log('req2\'s result: ', req2.result);
       }
   }
   ```

   `avtStore.add('asfdfsdfd')`会执行成功，但是开始等待Promise类实例settle的过程中会启动`setTimeout`调度。当`setTimeout`调度启动时，事务会被关闭，执行下面的`avtStore.add({name: 'sfsd', age: 12})`时会抛出TransactionInactiveError错误类实例。

## 错误处理

IndexedDB中的错误处理有2种方式：

1. 为数据操作请求添加error处理程序
   - 默认情况下，失败的数据操作请求将自动中止事务，并取消事务内的数据操作对数据库所做的所有更改。
   - 如果需要在数据操作请求失败时不让事务自动中止，可以在error处理程序中调用`event.preventDefault()`。
2. 事件委托
   - IndexedDB中事件的冒泡顺序：数据操作请求->事务->数据库。
   - 可以在事务或者数据库上添加error处理程序来统一捕获和处理下面的数据操作请求中产生的错误，在error处理程序内部可以通过`event.target`获取产生错误的数据操作请求。
   - 如果已经在数据操作请求层面捕获了事件且不想再让事件往上冒泡，可以调用`event.stopPropagation()`。

## 搜索

1. 在IndexedDB中，对对象库的搜索有2种方式。
   1. 通过对象库对象进行搜索，使用键字段的值或者键字段的值范围作为搜索条件
   2. 通过索引对象进行搜索，使用索引字段的值或者索引字段的值范围作为搜索条件

2. 在IndexedDB中，值范围通过IDBKeyRange类实例表达。IDBKeyRange类提供了4个静态方法来创建类实例：
   1. `IDBKeyRange.lowerBound(value[,open])` >=value的值范围。若open为true，则为>value的值范围。
   2. `IDBKeyRange.upperBound(value[, open])` <=value的值范围。若open为true，则为<value的值范围。
   3. `IDBKeyRange.bound(lowerValue, upperValue, [lowerOpen], [upperOpen])` lowerValue <= x <= upperValue的值范围。
   4. `IDBKeyRange.only(value)` 值范围只包含value。

3. 在IndexedDB中，和搜索相关的方法有5个，它们既可以通过对象库对象进行调用，也可以通过索引对象进行调用。
   1. `obj.get(query)` 返回第一个满足给定搜索条件的值
   2. `obj.getAll(query[, count])` 返回由所有满足给定搜索条件的值组成的数组。如果指定count，则使用count对结果数组的元素个数进行限制。
   3. `obj.getKey(query)` 返回第一个满足给定搜索条件的值关联的键
   4. `obj.getAllKey(query[, count])` 返回由所有满足给定搜索条件的值关联的键组成的数组。如果指定count，则使用count对结果数组的元素个数进行限制。
   5. `count` 返回满足给定搜索条件的值的个数

4. 创建索引对象

   - 索引对象是附加在对象库上的数据结构，创建索引对象是对对象库结构的修改，只能在`upgradeneeded`处理程序中进行。

   - 语法

     ```javascript
     objectStore.createIndex(indexName, keyPath[, option])
     ```

     `indexName`: 字符串，获取索引对象时会用到，相当于索引的id

     `keyPath`: 索引的对象字段的路径，如'price', 'profile.name'等

     `option`: 可选，如果提供，需要给一个包含unique和（或）multiEntry属性的对象。

     如果提供了`{unique: true}`，则需要保证添加的记录中的索引字段的值在对象库中是唯一的，否则操作会报错；如果提供了`{multiEntry: true}`，且索引字段的值是一个数组，则在对象库内部，会以所有数组中的每一个元素作为key保存一个值列表。

     unique属性会影响记录添加操作，multiEntry属性会影响搜索结果。

5. 获取索引对象

   - 语法

     ```javascript
     let indexObject = objectStore.index(indexName)
     ```

6. 使用索引对象进行搜索的示例代码

   ```javascript
   let db;
   let openRequest = indexedDB.open('foo', 11);
   
   openRequest.onupgradeneeded = function() {
       db = openRequest.result;
       
       if(db.objectStoreNames.contains('books')) {
           db.deleteObjectStore('books');
           console.log('delete books object store');
       }
       
       console.log('ready to create books object store');
       const books = db.createObjectStore('books', {keyPath: 'id'});
       // 创建3个索引
       books.createIndex('price_unique_idx', 'price', {unique: true});
       books.createIndex('tags_no_multi_entry_idx', 'tags', {multiEntry: false});
       books.createIndex('tags_multi_entry_idx', 'tags', {multiEntry: true});
   }
   
   openRequest.onsuccess = function() {
       console.log('open success');
       
       db = openRequest.result;
       db.onversionchange = function() {
           db.close();
           alert('Database is outdated, please reload  the page.')
       }
       
       const transaction1 = db.transaction('books', 'readwrite');
       transaction1.onerror = function(e) {
           console.log(e.target);
       }
   	const books = transaction1.objectStore('books');
       const req1 = books.add({
           id: "1",
           title: "Thinking, Fast and Slow",
           author: "Daniel Kahneman",
           publisher: "Farrar, Straus and Giroux",
           firstPublished: 2011,
           price: 100,
           tags: ["economics", "psychology"]
       });
       req1.onsuccess = function() {
           console.log('req1', req1.result);
       }
       req1.onerror = function(e) {
           e.preventDefault();
       }
       
       const req2 = books.add({
           id: "2",
           title: "Evicted: Poverty and Profit in the American City",
           author: "Matthew Desmond",
           publisher: "Crown",
           firstPublished: 2016,
           price: 100,
           tags: ["sociological", "politics", "ethnography"]
       });
       req2.onsuccess = function() {
           console.log('req2', req2.result);
       }
       req2.onerror = function(e) {
           // Unable to add key to index 'price_unique_idx': at least one key does not satisfy the uniqueness requirements.
           e.preventDefault();
       }
       
       const req3 = books.get('1');
       req3.onsuccess = function() {
           console.log('req3', req3.result);
       }
       
       const transaction2 = db.transaction('books', 'readwrite');
       transaction2.onerror = function(e) {
           console.log(e.target);
       }
       const tagsNoMultiEntryIndex = transaction2.objectStore('books').index('tags_no_multi_entry_idx');
       
       const req4 = tagsNoMultiEntryIndex.get(["economics", "psychology"]);
       req4.onsuccess = function() {
           console.log('req4', req4.result); // 有值
       }
       req4.onerror = function(e) {
           e.preventDefault();
       }
       
       const req5 = tagsNoMultiEntryIndex.get('economics');
       req5.onsuccess = function() {
           console.log('req5', req5.result); // undefined
       }
       req5.onerror = function(e) {
           e.preventDefault();
       }
       
       const transaction3 = db.transaction('books', 'readwrite');
       transaction3.onerror = function(e) {
           console.log(e.target);
       }
       const tagsMultiEntryIndex = transaction3.objectStore('books').index('tags_multi_entry_idx');
       
       const req6 = tagsMultiEntryIndex.get(["economics", "psychology"]);
       req6.onsuccess = function() {
           console.log('req6', req6.result); // undefined
       }
       req6.onerror = function(e) {
           e.preventDefault();
       }
       
       const req7 = tagsMultiEntryIndex.get(IDBKeyRange.lowerBound('economics'));
       req7.onsuccess = function() {
           console.log('req7', req7.result); // 有值
       }
       req7.onerror = function(e) {
           e.preventDefault();
       }
   }
   
   const bookRecommendations = [
       {
           title: "Thinking, Fast and Slow",
           author: "Daniel Kahneman",
           publisher: "Farrar, Straus and Giroux",
           firstPublished: 2011,
           price: 100,
           tags: ["economics", "psychology"]
       },
       {
           title: "Mindset: The New Psychology of Success",
           author: "Carol S. Dweck",
           publisher: "Ballantine Books",
           firstPublished: 2006,
           price: 110,
           tags: ["growth", "psychology"]
       },
       {
           title: "Guns, Germs, and Steel: The Fates of Human Societies",
           author: "Jared Diamond",
           publisher: "W.W. Norton & Company",
           firstPublished: 1997,
           price: 120,
           tags: ["anthropology", "history", "geography"]
       },
       {
           title: "Evicted: Poverty and Profit in the American City",
           author: "Matthew Desmond",
           publisher: "Crown",
           firstPublished: 2016,
           price: 100,
           tags: ["sociological", "politics", "ethnography"]
       },
       {
           title: "A Brief History of Time",
           author: "Stephen Hawking",
           publisher: "Bantam Books",
           firstPublished: 1988,
           price: 130,
           tags: ["cosmology", "black holes", "quantum", "science"]
       },
       {
           title: "The Order of Time",
           author: "Carlo Rovelli",
           publisher: "Riverhead Books",
           firstPublished: 2018,
           price: 80,
           tags: ["science", "physics", "thermodynamics", "relativity", "philosophy"]
       },
       {
           title: "The Pragmatic Programmer: Your Journey to Mastery",
           author: "Andrew Hunt, David Thomas",
           publisher: "Addison-Wesley Professional",
           firstPublished: 1999,
           price: 120,
           tags: ["software", "engineering", "programming", "technology"]
       },
       {
           title: "Clean Code: A Handbook of Agile Software Craftsmanship",
           author: "Robert C. Martin",
           publisher: "Prentice Hall",
           firstPublished: 2008,
           price: 100,
           tags: ["programming", "refactoring", "engineering", "software"]
       }
   ];
   ```

   

## 从对象库中删除

1. `objectStore.delete(key/IDBKeyRange)` 根据给定键字段的值或键字段的值范围从指定对象库中删除值
2. `objectStore.clear()` 清空指定对象库

## 光标（Cursor）

1. 光标（Cursor）对象是一种特殊的对象，它使用给定条件对对象库进行查询，但每次只返回一个键/值，每个结果触发一次success事件。
   这样可以减少查询数据库时的内存负担。

2. 可以通过对象库对象或对象库上的索引对象调用`openCursor`使用光标对象进行搜索：
   const request = objectStore.openCursor(键字段的值或以IDBKeyRange实例表达的键字段的值范围, [direction])
   
   const request = indexObj.openCursor(索引字段的值或以IDBKeyRange实例表达的索引字段的值范围, [direction])

   direction可选，可以传'next', 'prev', 'nextunique'和'prevunique'。
   request.onsuccess = () => { const cursor = request.result; ...}
   
3. 在搜索请求的onsuccess处理程序中，可以通过request.result获取到光标对象。

4. 光标对象的主要方法有：
   cursor.advance(count);
   cursor.continue([key]);

## Promise包装器

```javascript
let db;
try {
  db = await idb.openDb(name, version, (db) => {...})
}
catch(err) {
}

try {
 事务、请求
}
catch(err) {
}
```

1. 没有使用try..catch块捕获的错误将被window对象捕获：
   window.addEventListener('unhandledrejection', event => {
    let request = event.target;
    let error = event.reason; // let error = request.error;
    ...
   })
2. 如果需要在使用Promise包装器时获取到本地请求对象，可以像下面这么做：
   const promise = bookStore.add(obj);
   const request = promise.request; // 获取到本地请求对象
   const transaction = request.transaction; // 获取到本地事务对象
   ...
   const result = await promise; // 这样获取请求的结果