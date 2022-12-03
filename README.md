# linq
A place to record a few linq tips and techniques I use to make my linq/EF queries run faster

## Stopwatch first
For all of these techniques, they could be faster or slower - it's best to use a stopwatch around the code before and after to check that something has actually improved rather than improved in theory.
Note: does not literally need to be System.Diagnostics.Stopwatch, could be something like the github scientist pattern.
For example - EF lazy loading may be faster, or may be slower, it depends on how the data is stored in the database and how it's then used once in C#.  Note that when doing this testing, there are confounding variables
that should be considered:
1. Warm up time - is the code being compiled as you hit it for the first time?  Consider discarding the first run of a perf testing
2. DataContext has cached items - do you dispose the datacontext between experiments?  If not, the second run of a test may get a performance boost by accident

## ToList in the right place - not too early
TODO

## Lazy Loading
Lazy loading is a common performance problem, particularly when binding items in a list (list or table or repeater or similar)
For example, I have a navigation menu structure:
```SQL
CREATE TABLE MenuIcon (
    [MenuIconID] int,
    [FontAwesomeGlyph] char(200),
    PRIMARY KEY (MenuIconID)
);

INSERT INTO MenuIcon
    (MenuIconID, FontAwesomeGlyph)
VALUES
    (1, 'fa-home'),
    (2, 'fa-questionmark'),
    (3, 'fa-email')
    
CREATE TABLE Navigation (
    [NavigationID] int,
    [MenuText] char(200),
    [MenuIconID] int
    PRIMARY KEY (NavigationID),
    FOREIGN KEY (MenuIconID) REFERENCES MenuIcon(MenuIconID)
);

INSERT INTO Navigation
    ([NavigationID], [MenuText], [MenuIconId])
VALUES
    (1, 'Home', 1),
    (2, 'About', 2),
    (3, 'Contact Sales',3),
    (4, 'Contact Support',3)
;
```
The Navigation class will look something like:
```CSharp
public partial class Navigation {
	public int NavigationID { get; set; }
	public string MenuText { get; set; }
	public int MenuIconID { get; set; }
	
	public virtual MenuIcon MenuIcon { get; set; }
}
```
When I use the MenuIcon property, with Lazy loading turned on in EF, I get the menu icon.  This is because the virtual property is overridden at runtime by EF, so rather than getting an instance of a Navigation class
I actually get a dynamic proxy class which inherits from Navigation.  If you were to look at the implementation of the dynamic class, it'd look something like:
```CSharp
public partial class DynamicProxyToNavigation {
	public int NavigationID { get; set; }
	public string MenuText { get; set; }
	public int MenuIconID { get; set; }
	
	DataContext DC;
	public override MenuIcon MenuIcon {
		get { 
			return DC.MenuIcons.First(x => x.MenuIconID == MenuIconID);
		}
		set { /*Not shown for brevity*/ }
	}
}
```

So when I bind my navigation items into a list on my UI, I do something like:
```CSharp
var allNavigationItems = DataContext.Navigations.ToList();
foreach(var navigationItem in allNavigationItems) {
  NavListOnUI.AddItem(navigationItem.MenuText, navigationItem.MenuIcon.FontAwesomeGlyph);
}
```
This will effectively run multiple queries:
1. `SELECT * FROM Navigation` (because we ran DataContext.Navigations.ToList())
2. `SELECT * FROM MenuIcon WHERE MenuIconID = 1` (on the first go through the loop we invoke navigationItem.MenuIcon, which calls the virtual property which invokes this query)
3. `SELECT * FROM MenuIcon WHERE MenuIconID = 2`
4. `SELECT * FROM MenuIcon WHERE MenuIconID = 3`
5. `SELECT * FROM MenuIcon WHERE MenuIconID = 3` (may or may not run[^1])

[^1]: Settings on your data context, and your version of EF may change this behaviour. It's possible that EF will recognise you already did this and not do it again, or it may run the query
  but use its inbuilt change tracker to reduce time marshalling the data into C# (a query in EF if affected both by the speed to execute the sql and get the data back to the server, and also
  how much time it takes to take those records and populate your C# objects - getting the data from the database tables to C# objects can generically be called marshalling, though the term is not 
  exclusive to working with databases)

Running lots of queries is usually, but not always, worse than running fewer queries.

### Fix: Use Appropriate Includes
The simplest possible way to fix this would be to use an include statement, which tells EF to get the data at the same time (broadly "hey EF, do a left join here because I know I'll want the data"):
```CSharp
var allNavigationItems = DataContext.Navigations.Include(x => x.MenuIcon).ToList();
foreach(var navigationItem in allNavigationItems) {
  NavListOnUI.AddItem(navigationItem.MenuText, navigationItem.MenuIcon.FontAwesomeGlyph);
}
```

This will now run SQL something like `SELECT * FROM Navigation LEFT JOIN MenuIcon ON Navigation.MenuIconID = MenuIcon.MenuIconID` 

#### Possible Issues
In this example, the tables don't have lots of columns, so this is probably an appropriate improvement.
However, watch out for cartesian explosions (see below) and repeated data which may mean the result set is far larger than before.
If I run `SELECT * FROM Navigation` I get 4 records and 3 columns of data made up of two ints and a char(200).  Each record is 4+200+4 bytes, so about 832 bytes.
If I run `SELECT * FROM MenuIcon` I get an int and a char(200) and 3 records so 612 bytes.  Adding these together gives 1444 bytes.
However, if I run `SELECT * FROM Navigation LEFT JOIN MenuIcon ON Navigation.MenuIconID = MenuIcon.MenuIconID` I get:
| NavigationID  | MenuText        | MenuIconID | MenuIconID | FontAwesomeGlyph |
| ------------- | --------------- | ---------- | ---------- | ---------------- |
| 1             | Home            | 1          | 1          | fa-home          |
| 2             | About           | 2          | 2          | fa-questionmark  |
| 3             | Contact Sales   | 3          | 3          | fa-email         |
| 4             | Contact Support | 3          | 3          | fa-email         |

The row size is 2+200+2+2+200, and there are 4 rows, so this is 1624 bytes.  I think we can safely say that the extra 180 bytes aren't going to be a problem in this over simplified example,
but as you have wider tables and bigger data sets this becomes more of a consideration (especially if your database is hosted physically apart from your C# code - you may need to consider physical network speed at a certain level)
This problem increases exponentially when you add more left joins - notice the repeated data to get menu item 3 - the more joins in a query, the more repeated data you are likely to get.  This doesn't just impact the 
query speed and amount of data across the network, EF will (depending on settings) try to reconcile which records it's seen before as well, which can add to the execution time when serialising into C# objects.
When this gets bad enough, it's called a cartesian explosion.

### Cartesian Explosions
TODO


## Use Projections
TODO
