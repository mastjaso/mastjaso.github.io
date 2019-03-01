---
published: false
---

Lately I've been trying to write higher performance code by trying to make better use of Dictionaries and HashSets in C#. I've always known that they're a little slower to write to, and much faster to read from, but I didn't quite grasp the sheer performance gape until I was doing some research on HashSets and came across this StackOverflow post:
https://stackoverflow.com/a/10348367/8333865


The main takeaways can probably be pretty immediately gleaned from these three graphs:
**Add 1000000 objects (without checking duplicates):**
![Add Tests]({{site.baseurl}}/https://i.stack.imgur.com/BPz30.png)

**Contains check for half the objects of a collection of 10000:**
![Contains Tests]({{site.baseurl}}/https://i.stack.imgur.com/g8NTg.png)

**Remove half the objects of a collection of 10000:**
![Add Tests]({{site.baseurl}}/https://i.stack.imgur.com/MorzW.png)

With the above contains tests you can't even really tell just how much faster the dicionary is because it's below the margin of error. 

That being said, I often find Dictionaries cumbersome to work with in C#, I feel like they often clog up your code, making it more difficult to read, and they're more difficult to use as quickly as say, lists, especially once you've started adjusting to LINQ. 

As an added wrinkle, if you're interfacing with software like Revit through an API, there are many instances where you may want to gather, process, and index a bunch of information from the model when your application launches so that you can then very quickly query and look it up again throughout the rest of execution, especially if you have to deal with synchronizing data between Revit and an external database.

So to create some of these indexed models that are convenient for extremely rapid lookup, I started writing some extension methods to make dealing with dictionaries and converting between collections and dictionaries a little easier.

The first is **AddByKey**. This comes in handy if you've got any kind of enumerable collection of objects, and you want to add all of them to a dictionary, maybe using a uniqueID property as they key or something like that. Lists have the handy AddRange() function but there's no equivalent for dictionaries, and it would seem difficult since how would you pass in the key? 

Well LINQ saves the day by taking your collection of objects as an argument, and a LINQ function as the second that gets applied to that collection. So your calling code would look something like:
	
    var objectList = new List<myObject>() { new myObject("UniqueID-1"), new myObject("UniqueID-2") };
	var dict = new Dictionary<string, myObject>();
    
    dict.AddByKey(objectList, item => item.UniqueID);
    
Which in my mind is a super clear, and readable way of specifying what function you want applied to  each item in the list to be added. But this is what the extension method looks like:    

   public static void AddByKey<TKey, TValue>(this Dictionary<TKey, TValue> dict, IEnumerable<TValue> targets, Expression<Func<TValue, TKey>> propertyToAdd)
          {
              MemberExpression expr = (MemberExpression)propertyToAdd.Body;
              PropertyInfo prop = (PropertyInfo)expr.Member;

              foreach (var target in targets)
              {
                  var value = prop.GetValue(target);
                  if (!(value is TKey))
                      throw new Exception("Value type does not match the key type.");//shouldn't happen.
                  dict.Add((TKey)value, target);
              }
          }
 
Similarly, I ran into an issue where the Revit API gave me a list of RevitFamilies, and I wanted to wrap them in a little FamilyWrapper class that would pull some info and add some convenience methods. I also wanted all of these wrappers stored in a dictionary with the original family unique ID as the key so that I could immediately retrieve one of these already created families from the master list at a later point. 

So I made an extension method for any IEnumerable that will convert it to a dictionary based on two LINQ functions, creatively called, **ToDictionaryByFunctions**. Calling it would look something like this:
  
	List<RevitFamily> familyList = document.GetFamilies();
	Dictionary<string,RevitFamilyWrapper> dict = objectList.ToDictionaryByFunctions(x => x.UniqueId, x => new RevitFamilyWrapper(x));
  
And that's it, not your collection is in a dictionary based on whatever transformations you want to apply to each item through LINQ. This is the code of the extension method:

  public static Dictionary<TKey, TValue> ToDictionaryByFunctions<TKey, TValue, TListItem>(this IEnumerable<TListItem> inList, Expression<Func<TListItem, TKey>> keyToAdd,
              Expression<Func<TListItem, TValue>> valueToAdd)
          {
              var outDict = new Dictionary<TKey, TValue>();
              MemberExpression keyExpr = (MemberExpression)keyToAdd.Body;
              PropertyInfo     keyProp = (PropertyInfo)keyExpr.Member;
              MemberExpression valueExpr = (MemberExpression)valueToAdd.Body;
              PropertyInfo     valueProp = (PropertyInfo)valueExpr.Member;

              foreach (var target in inList)
              {
                  var tempKey = keyProp.GetValue(target);
                  if (!(tempKey is TKey))
                      throw new Exception("Key type does not match the correct type.");//shouldn't happen.
                  var tempValue = valueProp.GetValue(target);
                  if (!(tempValue is TValue))
                      throw new Exception("Value type does not match the correct type.");//shouldn't happen.
                  outDict.Add((TKey)tempKey, (TValue)tempValue);
              }
              return outDict;
          }
  
  