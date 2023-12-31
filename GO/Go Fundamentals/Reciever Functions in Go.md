Code for refernce

```Go
type model struct {
    choices  []string         // items on the list
    cursor   int              // which item our cursos is at
    selected map[int]struct{} // which items are selected
}

func (m model) Init() tea.Cmd { 
	return nil 
}
```

In Go, a [[Fundamental]] syntax `(m model)` is called a "receiver" and it is used to define a method on a type. In this case, `m` is the name of the receiver and `model` is the type of the receiver.

The receiver is like a parameter that is implicitly passed to the method when it is called. It is often used to access or modify the fields of the type, or to call other methods on the same type.

Here is an example of how you might define a method with a receiver in Go:

```GO
type MyType struct {
  // fields go here
}

func (m MyType) MyMethod() {
  // method implementation goes here
  // you can access the fields of MyType using m.fieldName
}
```

In this example, `MyMethod` is a method on the `MyType` type. It has a receiver named `m` of type `MyType`, which allows you to access the fields of the type and call other methods on the same type within the method's implementation.

Thanks to ChatGPT