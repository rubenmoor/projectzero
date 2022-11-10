In case you have some property of type `FProperty` and you need its value:

```
float NewVelocity = FFloatProperty::GetPropertyValue(SomeProperty->ContainerPtrToValuePtr<void>(this));
```

This is for a property of type `float`.
Replace `FFloatProperty` for the type of your property.
`this` is the `UObject` that has the property `SomeProperty`.
