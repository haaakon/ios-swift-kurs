# PG5600
# iOS programmering
## Forelesning 8 - Persistering

---

# Sist gang

* Debugging
* Testing
* Swift og gjenbruk av kode
  * Rammeverk
  * Cocoapods & Carthage
* Tråder og asynkronitet
* Snakke med internett
* try & json

---

# Agenda

* Hvor filer lagres
* Enkel lesing og skriving til disk
* UserDefaults
* NSKeyedArchiver / NSKeyedUnarchiver
* Core Data
* Keychain

---

# Hvor filer lagres

```swift
let fm = FileManager.default()

// <app home>/Documents, backes opp, kan bli vist til bruker
// For brukerens datafiler
print(fm.URLsForDirectory(.DocumentDirectory, inDomains: .UserDomainMask)[0])

// <app home>/Library, backes opp, skjult
// For det som ikke er brukerens datafiler
print(fm.URLsForDirectory(.LibraryDirectory, inDomains: .UserDomainMask)[0])

// <app home>/Library/Caches, backes IKKE opp, skjult
// F.eks. for caching av bilder
print(fm.URLsForDirectory(.CachesDirectory, inDomains: .UserDomainMask)[0])

// <app home>/tmp, backes IKKE opp
// For midlertidige filer som ikke trenger å eksistere mellom launches
print(TemporaryDirectory())
```

---

# Skriv og les string til disk

```swift
   let dir = FileManager.default().URLsForDirectory(.DocumentDirectory,
            inDomains: .UserDomainMask)[0] as! URL

   let string: NSString = "Hello world"

   let path = dir.URLByAppendingPathComponent("file.txt").path!

   try! string.writeToFile(path, atomically: true, encoding:NSUTF8StringEncoding)

   let savedString = try! NSString(contentsOfFile: path, encoding: NSUTF8StringEncoding)

```

---

# Skriv og les NSDictionary til plist

```swift
let dir = NSFileManager.defaultManager().URLsForDirectory(.DocumentDirectory,
				inDomains: .UserDomainMask)[0] as NSURL
let dict = ["workouts": 23] as NSDictionary
print(dir)

let path = dir.URLByAppendingPathComponent("file.plist").path!

// Skriv
dict.writeToFile(path, atomically: true)

// Les
print(NSDictionary(contentsOfFile: path))
```

---

# NSUserDefaults

---

# NSUserDefaults

* For enkle verdier og caser
* Eksempel: lagre brukerens preferanser
* Verdier caches slik at man slipper diskaksess ved hver henting
* Synkronisering av verdier mot disk skjer automatisk ved jevne mellomrom

---

```swift
let userDefaults = NSUserDefaults.standardUserDefaults()
userDefaults.setObject("Tim Cook", forKey: "name")

if let name = userDefaults.stringForKey("name") {
    print("Got Name: \(name)")
}

// boolForKey, intForKey .. osv

// Tving synkronisering mot disk
userDefaults.synchronize()
```

---

# NSKeyedArchiver / NSKeyedUnarchiver

---

# NSKeyedArchiver / NSKeyedUnarchiver

* For serialisering av objekter til disk
* Klasser som skal serialiseres må implementere NSCoding-protokollen. Mange standard datatyper gjør dette allerede.
* [Pass på fremover og bakoverkompatibilitet](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/Archiving/Articles/compatibility.html)

---

# Klasse som implementerer NSCoding

```swift

class Workout : NSObject, NSCoding {
    var name: String!
    var entries: Int = 0

    required convenience init(coder aDecoder: NSCoder) {
        self.init()
        name = aDecoder.decodeObjectForKey("name") as! String
        entries = aDecoder.decodeIntegerForKey("entries")
    }

    func encodeWithCoder(aCoder: NSCoder) {
        aCoder.encodeObject(self.name, forKey: "name")
        aCoder.encodeInteger(self.entries, forKey: "entries")
    }
}
```

---

# Eksempel serialisering og deserialisering

```swift
let fm = NSFileManager.defaultManager()

let libDir = fm.URLsForDirectory(.LibraryDirectory, inDomains: .UserDomainMask)[0] as NSURL
print(libDir)

let workout = Workout()
workout.entries = 14
workout.name = "Pull-ups"

let path = libDir.URLByAppendingPathComponent("workouts").path!

// Serialisere til disk
NSKeyedArchiver.archiveRootObject(workout, toFile: path)

// Deserialisere fra disk
let savedWorkout = NSKeyedUnarchiver.unarchiveObjectWithFile(path) as! Workout
print("\(savedWorkout.name), entries: \(savedWorkout.entries)")
```

---

# Core Data

---

# Core Data

For mer komplekse behov, når du trenger:

* Å lagre objektgrafer (med relasjoner)
* Å gjøre spørringer mot objektgrafer
* Støtte undo/redo
* Lagre objektgrafer i iCloud

---

![](img/nshipster1.png)


---

# Terminologi

* **Managed Object Context** - "hovedobjektet" mot Core Data. Håndterer et sett Managed Objects og deres lifssyklus
* **Persistent Store Coordinator** - abstraherer bort underliggende store. Implementerer henting/lagring/sletting/m.m. av MOM mot store
* **Managed Object Model** - som et slags databaseskjema
* **Persistent Object Store** - XML, SQLite, Binær, custom

![fit right](img/coredata.png)

---

# ManagedObjectContext

```swift
lazy var managedObjectContext: NSManagedObjectContext? = {
    let coordinator = self.persistentStoreCoordinator
    if coordinator == nil {
        return nil
    }
    var managedObjectContext = NSManagedObjectContext()
    managedObjectContext.persistentStoreCoordinator = coordinator
    return managedObjectContext
}()
```

---


# ManagedObjectModel

```swift
lazy var managedObjectModel: NSManagedObjectModel = {
    let modelURL = NSBundle.mainBundle().URLForResource("Workouts",
    	withExtension: "momd")!
    return NSManagedObjectModel(contentsOfURL: modelURL)!
}()

```

---

# PersistentStoreCoordinator

```swift
    lazy var persistentStoreCoordinator: NSPersistentStoreCoordinator = {

        // Create the coordinator and store
        let coordinator = NSPersistentStoreCoordinator(managedObjectModel: self.managedObjectModel)
        let url = self.applicationDocumentsDirectory.URLByAppendingPathComponent("SingleViewCoreData.sqlite")

      	let coordinator = the try! coordinator.addPersistentStoreWithType(NSSQLiteStoreType, configuration: nil, URL: url, options: nil)
      	return coordinator
     }
```


---

# DEMO

### single view application - Check Core Data

---

# Managed Objects

![inline](img/managedobject.png)

I utgangspunktet NSManagedObject's med key/value. Disse **kan** og **bør** ofte ha en subklasse for renere kode og mer funksjonalitet, men må ikke

---

# Fetch Requests

![inline](img/fetchrequest.png)

---

# Opprett

```swift
let entity = NSEntityDescription.entityForName("Workout",
				inManagedObjectContext: moc)

var workout = Workout(entity: entity!,
				insertIntoManagedObjectContext: moc)
workout.name = excersice
workout.entries = 0


try! moc.save()

workouts.append(workout)
```

---
# Opprett med convenience init


```swift

// nilable initializer!
 convenience init?(attributes: [String : String], managedObjectContext: NSManagedObjectContext) {

        self.init(entity: NSEntityDescription.entityForName("Workout", inManagedObjectContext: managedObjectContext)!, insertIntoManagedObjectContext: managedObjectContext)


     	if let actualName = attributes["name"] as? String {
        	self.name = actualName
        } else {
            return nil
        }

        if let actualDifficulty = attributes["difficulty"] as? Int {
         	self.difficulty = actualDifficulty
        } else {
         	return nil
        }

    }

 ```
---

# Hent alle

```swift

let query = NSFetchRequest<Workout>(entityName: "Workout")

let results = try! moc.fetch(query)
workouts = results
    //  tableView.reloadData()

```

---

# Fetch med predicate (spørring)

```swift
var query = NSFetchRequest<Workout>(entityName: "Workout")
query.predicate = NSPredicate(format: "entries >= %d", 5)
// workout sin attributt entries

let results = try! moc.fetch(query)
    workouts = results
    //tableView.reloadData()

```

---

# Sortering og limit

```swift

// Hent topp 3
var fetchRequest = NSFetchRequest<Workout>(entityName: "Workout")
fetchRequest.sortDescriptors = [NSSortDescriptor(key: "entries", ascending: false)]
fetchRequest.fetchLimit = 3

```

---



# Endre

```swift
//let workout = workouts[indexPath.row]

workout.entries = workout.entries.integerValue + 1

var error: NSError?
try! moc.save()

```

---

# Slett

```swift

//let workoutToRemove = workouts[indexPath.row]
moc.deleteObject(workoutToRemove)
try! moc.save()

```

---

# Telle

```swift
let fetchRequest = NSFetchRequest(entityName: "Workout")
// her må du bruke error
var error : NSError?

let count = moc.countForFetchRequest(fetchRequest, error: &error)

if error != nil {
    print("error skjedde!: \(error)")
}
print("\(count) øvelser registrert")
```

---

# NSFetchedResultsController

* Istedet for `tableView.reloadData()`!
* Går hånd i hanske med UITableView
* Gir gruppering, caching, synkronisering av tableview mot data
* Abstraksjon rundt fetchRequest og resultatene
* Må ha minst en sortDescriptor
* Rask visning av mye data i TableView

---

# NSFetchedResultsController API

```swift

// Instansier (eks. i viewDidLoad)
NSFetchedResultsController(fetchRequest: query,
        managedObjectContext: moc, sectionNameKeyPath: nil, cacheName: nil)

// Hent data
fetchedResultsController.performFetch(&error)

// Få antall rader i section:
fetchedResultsController.sections![section].numberOfObjects

// Hent ut objekt med:
fetchedResultsController.objectAtIndexPath(indexPath)
```

---

# NSFetchedResultsController - automatisk oppdatering av tableview

```swift

class ViewController: UIViewController, UITableViewDataSource,
    UITableViewDelegate,NSFetchedResultsControllerDelegate {

    // I ViewDidLoad, new opp NSFetchedResultsController og sett delegate:
    // fetchedResultsController.delegate = self

    func controllerWillChangeContent(controller: NSFetchedResultsController) {
        tableView.beginUpdates()
    }

    func controllerDidChangeContent(controller: NSFetchedResultsController) {
        tableView.endUpdates()
    }
}
```

---

```swift
func controller(controller: NSFetchedResultsController,
    didChangeObject anObject: AnyObject,
    atIndexPath indexPath: NSIndexPath,
    forChangeType type: NSFetchedResultsChangeType,
    newIndexPath: NSIndexPath) {

    switch type {
        case .Insert:
            self.tableView.insertRowsAtIndexPaths([newIndexPath],
                withRowAnimation: UITableViewRowAnimation.Automatic)
        case .Update:
            if let cell = self.tableView.cellForRowAtIndexPath(indexPath) {
                self.configureCell(cell, indexPath: indexPath)
            }
        case .Move:
            tableView.deleteRowsAtIndexPaths([indexPath], withRowAnimation: .Automatic)
            tableView.insertRowsAtIndexPaths([newIndexPath], withRowAnimation: .Automatic)
        case .Delete:
            tableView.deleteRowsAtIndexPaths([indexPath], withRowAnimation: .Automatic)
        default:
            break
    }

}
```

---

# Keychain

* For lagring av sensitive data (passord m.m.)
* C API :'-(
* Se "iOS Keychain Services Tasks" i dokumentasjonen for mer info
* Typisk bra å bruke Cocoapods for https://cocoapods.org/pods/SwiftKeychain

---

---
# Videre lesning

Apples intro til CoreData:
 - https://developer.apple.com/library/watchos/documentation/Cocoa/Conceptual/CoreData/index.html
 - Kapittel 17 i Swift Cookbook
 - Forelesning 16 i Swiftkurset til Stanford

---


# Oppgaver

## Se Oppgaver på IT's learning


#### Forelesningen er basert på fjorårets foiler, laget av
#### Hans Magnus Inderberg og Mads Mobæk
