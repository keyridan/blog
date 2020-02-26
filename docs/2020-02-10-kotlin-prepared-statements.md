## Kotlin reflection and prepared statement tutorial 

![bmo and reflection](https://cdn-images-1.medium.com/max/1024/1*MUAWwshzUHcJjtj7Gm-rdA.png)

Recently I decided to write my [own habit tracker](https://github.com/j0rsa/bujo-tracker). In my project for access to the database, I opted for [exposed](https://github.com/JetBrains/Exposed). It’s a lightweight SQL library, which I find pretty amazing and easy to use. But at some point, I faced the need to write a bit more complex query, mainly to [calculate habit streaks](https://medium.com/@keyridan/sql-story-of-unbroken-chains-of-events-daily-weekly-streaks-f7f4c36e5bf6)on the database side. In that case [exposed FAQ](https://github.com/JetBrains/Exposed/wiki/FAQ#q-is-it-possible-to-use-native-sql--sql-as-a-string) has a native SQL solution, which you can implement on top of the library. It lets you execute queries and map its result. This improvement allows you to write your queries in such a manner:

{% gist https://gist.github.com/keyridan/05e841297efee0b39a6b167e06e40a07 %}

What can be a real problem here is an SQL injection since the code executes the whole string as SQL statement. It doesn’t allow you to separate parameters to use them in the context of parameters, not as a SQL command. That’s why I decided to go with an old good PreparedStatement. In this tutorial, I will try to improve its use with Kotlin type system. All code is available [over on GitHub](https://github.com/keyridan/preparedStatementTutorial), with one branch per chapter so that you can follow along.
* [Basic work with `Prepared Statement`](#basic) - master branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/master)
* [Adding user table](#userTable) - userTable branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/userTable)
* [Adding map for `ResultSet?`](#mapResult) - mapResult branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/mapResult)
* [Extract values with reflection](#reflect-getValue) - reflect-getValue branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/reflect-getValue)
* [Extract entities with reflection](#reflect-dataClass) - reflect-dataClass branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/reflect-dataClass)
* [Extract values using types](#types-getValue) - types-getValue branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/types-getValue)
* [Query parameters](#queryParameters) - types-getValue branch [here](https://github.com/keyridan/preparedStatementTutorial/tree/queryParameters)
* [Conclusion](#conclusion)

#### 1. Basic work with Prepared Statement — [master branch](https://github.com/keyridan/preparedStatementTutorial/tree/master) <a name="basic"></a>

Let me quickly introduce the base project. In the [gradle file](https://github.com/keyridan/preparedStatementTutorial/blob/master/build.gradle.kts) we have such libraries as `h2database`, `config4k` and `exposed` for database connection and transactions. For testing purposes, we added dependencies: `JUnit` and `assertk`.

{% gist https://gist.github.com/keyridan/e50fa58788c52cb15bb4c4316144da2c %}

File `AppConfig.kt` loads database configs from `application.conf`. `TransactionManager` gets database connection and provides us with the `currentTransaction`. Its method `tx` allows us to run queries within a transaction.

{% gist https://gist.github.com/keyridan/77de24dc5f57c28bf34d175dbb7ddf90 %}

The main focus of this tutorial is `Queries` and `QueriesKtTest`. There is the first version of native SQL executor:

{% gist https://gist.github.com/keyridan/1f5f8882dea0a9846abba51425399a64 %}

Nothing fancy here. It takes `currentTransaction`, creates `preparedStatement` from our string and executes it.   
Test for this method:

{% gist https://gist.github.com/keyridan/b2c0f47a08216ad18cd9b11170bc0234 %}

As you can see we have to work directly with the `ResultSet`, call `next()` to obtain record and parse string value of the column with `getString()` method. Let’s add a new table and see how it adds complexity to the code.

#### 2. Adding user table — [userTable branch](https://github.com/keyridan/preparedStatementTutorial/tree/userTable) <a name="userTable"></a>

Let’s quickly introduce a new table in terms of exposed library. For this purpose we’ll add `Tables.kt`:

{% gist https://gist.github.com/keyridan/63bba78d21cc9b331694bfdfd57a5a5f %}

Object `Users` represents table with two columns: `id: Int`, `name: Varchar`. Test in this branch is more complicated. Instead of `tx` method we will use a new one:

{% gist https://gist.github.com/keyridan/15fcd92c671499493d5554b9f4380c42 %}

It creates schema with our table and rollback transaction afterwards.

{% gist https://gist.github.com/keyridan/70ba42f746e2cf1c801ae3eadf185f13 %}

The test now inserts 2 user rows into the table and reads them one after another. Let’s create `map` for `ResultSet` to make parsing easier.

#### 3. Adding map for `ResultSet?` — [mapResult branch](https://github.com/keyridan/preparedStatementTutorial/tree/mapResult)<a name="mapResult"></a>

This sections contains just a few tweaks. Our test lost few lines of code:

{% gist https://gist.github.com/keyridan/a959ad62bd5ac27c0159e795896c6b58 %}

Now it directly calls `getValue`.

{% gist https://gist.github.com/keyridan/901b6b2f27edf543b8b1659070f761f3 %}

All `resultSet.next()` now can be found in the map method. It takes transform `ResultSet` as a function. In our case, it’s the same `getString` with the name of the column. Basically, `getValue` maps through the result set and takes its string values.  
Of course, not all our query result values are strings. Let’s use some reflection to expand the capabilities of our method.

#### 4. Extract values with reflection —[ reflect-getValue branch](https://github.com/keyridan/preparedStatementTutorial/tree/reflect-getValue) <a name="reflect-getValue"></a>

As soon as we don’t want to create tons of method for each type that we want to extract from the result set, we’ll use reflection.

{% gist https://gist.github.com/keyridan/8453d3afc5a842f5a6724538cf64e401 %}

Here we compare class we want to extract with some known classes. We call `::class` to obtain Kotlin `classKClass<T>` value of them. In the case of a match, we call the corresponding `ResultSet` method.  
The `toValue` function:

{% gist https://gist.github.com/keyridan/9277b8ac065bf1fa16b798d8c7e7c6a4 %}

Note that it also takes `kClass` parameter. But we don’t want to pass directly the class value. We’ll leave this part of work to Kotlin and use its magic of `inline fun` and `reified` [type parameteres](https://kotlinlang.org/docs/reference/inline-functions.html#reified-type-parameters).

{% gist https://gist.github.com/keyridan/dc316c81c17f16924c3f25b96a6661b2 %}

With reified we can access the type `T` and get class of it, the compiler knows the actual type used as a typed argument.   
Now we call `getValue` as `.getValue<String>(“name”)` or `getValue<Int>(“id”)`. In our case we don’t even need to change anything in our test:

{% gist https://gist.github.com/keyridan/942b0f58e878bc12525bd4aaf09f373a %}

The compiler has enough information about type `T` from the definition of the `resultSet` variable.  
Amazing! Let’s add some more reflection to be able to convert `ResultSet` to data class.

#### 5. Extract entities with reflection — [reflect-dataClass branch](https://github.com/keyridan/preparedStatementTutorial/tree/reflect-dataClass) <a name="reflect-dataClass"></a>

Let’s create data class for our `Users` table in `QueiesKtTest`.

{% gist https://gist.github.com/keyridan/9bcb19f2dc8ffa272e62872bbed9ca9b %}

How can we create a class with reflection? We can obtain information about the class constructor and its parameters.

{% gist https://gist.github.com/keyridan/ce34cb51930316df4898afddc4782734 %}

With the type and name of each parameter we extract values from the result set. And to create a class instance we call the constructor with all these values as vararg arguments. One more cool thing is how we determine the `KClass` values of the parameters. We use [reflection](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.jvm/jvm-erasure.html)[jvmErasure](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect.jvm/jvm-erasure.html):

> Returns the KClass instance representing the runtime class to which this type is erased to on JVM.

Here are another examples of `reified` usage to convert the result set to `data class`:

{% gist https://gist.github.com/keyridan/32770ed818ad8011c4ba876cf3559cc0 %}

Note that function can be `inline` only if it calls public functions. In this case, we can easily call one `inline` function from another one. Here’s the updated test:

{% gist https://gist.github.com/keyridan/f782abf23dc5bf68d49508549076af69 %}

We changed `getValue(“name”)` to `toEntities()`.

#### 6. Extract values using types types — [getValue branch](https://github.com/keyridan/preparedStatementTutorial/tree/types-getValue) <a name="types-getValue"></a>

Reflection is amazing, but all its magic is in runtime. While we would rather rely on type system. Let’s try rewrite `getValue`.  
First, we need to create classes that will represent our columns with types and names:

{% gist https://gist.github.com/keyridan/f936ffa237d0acd81e8579615d01e8b3 %}

Let’s repeat `extract` method in terms of these new types:

{% gist https://gist.github.com/keyridan/bc228a2407faf376f91c87e987a63dc2 %}

While we use `sealed` classes here we don’t need to use else branch, because compiler can prove that all possible cases are covered.   
Here with usual `inline` and `reified` you can use `infix` [notation](https://kotlinlang.org/docs/reference/functions.html#infix-notation). All these let us write code such as: `BooleanColumn(“name”) from result`

Let’s create a new test and use all our result conversion functions:

{% gist https://gist.github.com/keyridan/edc7c0a44741423543a1e1ca8555c9a9 %}

These time we use `row_number()` function in the query. It returns an integer value, the column name is `rn`. To parse the result we can use:

{% gist https://gist.github.com/keyridan/d7fb650dd2bfe6b8cda492e91d26bfdf %}

#### 7. Query parameters — [queryParameters branch](https://github.com/keyridan/preparedStatementTutorial/tree/queryParameters) <a name="queryParameters"></a>

We have a couple of options to extract values from the query result. But we still have no way to provide arguments to `prepared statement`. Let’s now focus on that.  
As in the last chapter let’s add some more types:

{% gist https://gist.github.com/keyridan/8cdb920fa6c12e366711919436e7b2d4 %}

And match through them:

{% gist https://gist.github.com/keyridan/956fc97b0422c8778d8442615f55abc2 %}

As soon as we can express our parameters, we need to update `exec` function to pass these values to the `prepared statement`:

{% gist https://gist.github.com/keyridan/68fe74d6939ec8032c995c5b31577cb0 %}

For each parameter we call its prepared statement set version and also pass its index, so don’t forget to keep the order of variables.  
Also we added a method to map through the result set with our previous chapter’s from function:

{% gist https://gist.github.com/keyridan/1d2df1635509b33e40a636c259a00e90 %}

Let’s check our improvements with a new test. Let’s say we want to get all records with names that are longer than some value and contain `"@"`.

{% gist https://gist.github.com/keyridan/091036d6f66b5658b392ebbec4e205bf %}

In this case, we’ll have only one name as a result.

#### Conclusion <a name="conclusion"></a>

From now on, we are prepared for huge queries that we couldn’t express with exposed:

- SQL injection safe
- the result can be easily converted to our types

As a final result, we have two conversion versions. One uses the reflection. You are required only to specify your type, it does all conversion by itself, but not type-safe. The second uses specified columns, it is type-safe, but all conversion descriptions are on you.  
Please, let me know which option you would prefer to use!
