# Protocol Buffer Basic Example

Protocol buffers allow you to efficiently (both in space and time) encode and decode complex messages.

It also allows you to define the message type in a convenient and general syntax, and then compile it into the language you wish to work with.  E.g., one could send a protobuf message from a C++ application and read it from a python application easily.

#### Basic Syntax

The syntax looks like this:

```proto
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = MOBILE];
  }

  repeated PhoneNumber phone = 4;
}
```

The numbers are unique identifiers for each field; each field must have one.

#### Compilation

If the above file were saved as `person.proto`, we could use the protobuf compiler to compile it via:

`protoc person.proto --cpp_out=.`

where the `--cpp_out=.` option indicates that we want to compile it into C++ and the output directory should be the current directory.  If instead we want to compile it into python, we could do:

`protoc person.proto --python_out=.`

#### Language Interface

Now you can simply import/include the compiled file into your application, and set/get fields via `set_X()` and `X()` respectively.  The other important action you can do is serialize/deserialze the message objects to/from bytestreams.

C++ example:

```c++
#include <iostream>
#include <fstream>
#include <string>

#include "person.pb.h"

int main() {
  const char * output_filename = "test_person";

  std::cout << "Creating person..." << std::endl;
  Person person;
  person.set_name("Apollys");
  person.set_id(1437);
  person.set_email("apollys@gmail.com");
  std::cout << "Name: " << person.name() << std::endl;
  std::cout << "ID: " << person.id() << std::endl;
  std::cout << "Email: " << person.email() << std::endl;
  std::fstream output(output_filename, std::ios::out | std::ios::binary);
  person.SerializeToOstream(&output);
  output.close();
  std::cout << std::endl << "Person saved to " << output_filename << std::endl;

  std::cout << std::endl << "Reading person..." << std::endl;
  std::fstream input(output_filename, std::ios::in | std::ios::binary);
  Person person2;
  person2.ParseFromIstream(&input);
  std::cout << "Name: " << person2.name() << std::endl;
  std::cout << "ID: " << person2.id() << std::endl;
  std::cout << "Email: " << person2.email() << std::endl;

  return 0;
}
```

The top half will create a Person object and serialize it to our output file, which yields something like this (except the browser can't necessarily display all the special characters):

```

Apollys�apollys@gmail.com
```

Then the second half of the program deserializes this file, creating a new Person object and demonstrates that we got back what we put in.  The output is:

```
Creating person...
Name: Apollys
ID: 1437
Email: apollys@gmail.com

Person saved to test_person

Reading person...
Name: Apollys
ID: 1437
Email: apollys@gmail.com
```

Note: in order to compile the final C++ app, I used: 

```
g++ main.cpp person.pb.cc -o app -lprotobuf
```
