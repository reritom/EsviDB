# Esvi

Esvi is a model interface that for easily adding new databases. Currently it supports sqlite3 and esvicore.
The model interface creates a clear seperation between the database and your flow

## How does it work?
In your code you can define Models by inheriting from a Model. Inside your definition you set the field names for your model, and their types, such as text fields, datetime fields, integer fields, and foreign keys. You can also explicitly set the model name, or implicitly use the class name as the model name.

### A model example
```
from esvi import model
from esvi import fields

class Contact(model.Model):
  first_name = fields.StringField(primary=True)
  surname = fields.StringField()
  age = fields.IntegerField()
  email_address = fields.StringField(default=None)
```

### Supported fields
Esvi supports four field types:
- IntegerField
- StringField
- ForeignKey
- JSONField
- ObjectField

Most of these are what you would expect. With the IntegerField and StringField supporting being primary keys. The foreign key is initialised with another Model. The JSONField is a glorified string field which shifts the responsibility of encoding and decoding to the adapter. The ObjectField is defined with an object class as the value. Then in instantiation, you pass the relevent class instance to it. The object class is required to have a `serialise` and `deserialise` which create a dict representation of the object, and reconstruct the object from the same dict representation. It also requires that the __init__ has all optional parameters.

### The ObjectField
The other fields are self explanatory, but the ObjectField requires a bit more explanation. The following is a simple example.

Consider a class, Car. The car can be instantiated perhaps with colour, size, max_speed, etc. Which would look like the following.

```
class Car():
  def __init__(self, colour, speed, size):
    self.speed = speed
    self.colour = colour
    self.size = size

  def set_speed(self, value):
    self.speed = value
```

To have this class useable in the ObjectField we would have to add the following:

```
class Car():
  def __init__(self, colour=None, speed=None, size=None):
    self.speed = speed
    self.colour = colour
    self.size = size

  def set_speed(self, value):
    self.speed = value

  def serialise(self):
    return {'speed': self.speed,
            'colour': self.colour,
            'size': self.size}

  def deserialise(self, bom):
    self.speed = bom.get('speed')
    self.colour = bom.get('colour')
    self.size = bom.get('size')

```
 And this would be then used as follows:

```
import Car
from esvi import model

class RandomModel(model.Model):
  random_value = fields.IntegerField()
  car_thing = fields.ObjectField(Car)
```

And could be created like so:
```
car = Car('Red', 15, 100)

new_random_model = RandomModel.create(random_value=20,
                                      car_thing=car)

# And you can then access the car attributes as normal
new_random_model.car_thing.set_speed(20)
```


In most cases, the ObjectField can be avoided by using a ForeignKey to another model. But sometimes the object may be too minor to warrant a new table or equivalent.

### Interacting with your model
Before interacting with your model, you need to set up your database connection.
For simplicitly, at the beginning of your flow, you will need to do the following:
```

from esvi.esvi_setup import EsviSetup

setup = EsviSetup()
setup.set_database_path(path_to_your_db)

# This creates a connection object accessible throughout the model interface, this is required
setup.set_global_connection()
```

Now, from inside your flow you can do the following:
### Create a new instance of your model
```
contact = Contact.create(first_name="Tony", surname="Stark", age=43)
```
This returns a ModelInstance object which has its own set of interactions.
The ModelInstance uses getters and setters to change its value, and then has an explicit save function for committing the changes to the database.
```
contact.set("age", 44)
contact.save()
```
The ModelInstance also supports dot operators for getting and setting the content.
```
contact.age = 15
contact.name = "Jack"
contact.save()
```
If you are done with a ModelInstance, you can delete it from the database using delete().
```
contact.delete()
```
If you attempt anything with this object now, you'll raise an InstanceDeletedException.

### Retrieving your models
There are two ways to retrieve a model. You can either retrieve by primary key, or you can filter.
When you retrieve by primary key, you just need to pass the value of your primary key to the retrieve method
```
contact = Contact.retrieve(primary_key="Tom")
```
This returns the typical ModelInstance object.

However, if you wish to filter or retrieve all, then multiple models can be returned, and are therefore returned in a class called the ModelSet.
This is simply an iterable which contains a list of ModelInstances.
```
# This is a ModelSet
all_contacts = Contact.retrieve_all()

for contact in all_contacts:
  # contact is a ModelInstance
  print(contact.get('name'))
```
The ModelSet has the method <i>exists()</i> which returns a boolean for if the ModelSet contains any ModelInstances.
This can be used for validation.
```
if Contact.retrieve_all().exists():
  # Do something
else:
  # We have no contacts, do something else
```

### Foreign keys
Esvi supports foreign keys.
```
from esvi import model
from esvi import fields
from test.test_models.contact import Contact
from test.test_models.message import Message

class Recipient(model.Model):
    recipient_id = fields.StringField(primary=True)
    address = fields.StringField()
    contact = fields.ForeignKey(Contact)
```
In this example we create a Recipient model which has a Contact foreign key.
To instantiate this model, we need to pass a contact ModelInstance (the same contact as the previous example).

```
recipient = Recipient.create(recipient_id="Something",
                             address="Something else",
                             contact=contact)
```
Now the contact can be accessed the same way as any value, and the contact attributes too.
```
recipient_contact = recipient.contact

recipient_contact_name = recipient.contact.name
recipient_contact_name = recipient.get('contact').get('name')
```


## Things to be implemented
- Esvicore adapter
- Specific Exceptions
