**关键概念** 
- com.dyuproject.protostuff.Schema 对应Class 
- com.dyuproject.protostuff.runtime.MappedSchema.Field 对应java.lang.reflect.Field 

如下几段伪代码说明了从一个java class构建protostuff schema的过程：   
```java 
com.dyuproject.protostuff.runtime.RuntimeSchema#createFrom 
  findInstanceFields:fieldsMap
  foreach field in fieldsMap 
    com.dyuproject.protostuff.runtime.RuntimeFieldFactory#getFieldFactory().create():com.dyuproject.protostuff.runtime.MappedSchema.Field
  new com.dyuproject.protostuff.runtime.RuntimeSchema(typeClass,  fields,int lastFieldNumber, instantiator)  
  
```  
```java 
com.dyuproject.protostuff.runtime.RuntimeFieldFactory#getFieldFactory(clazz, strategy)
  switch clazz  
    case xx return _RuntimeFieldFactory_CONST_INSTANCE 
```
```java
static final RuntimeFieldFactory<Long> INT64;
static {
if com.dyuproject.protostuff.runtime.RuntimeEnv#USE_SUN_MISC_UNSAFE 

  INT64 = RuntimeUnsafeFieldFactory.INT64;
else 
  INT64 = RuntimeReflectionFieldFactory.INT64;

```
注意后面这个USE_SUN_MISC_UNSAFE,protostuff根据jre环境判断，如果是sun jre 就会使用unsafe 否则使用 reflect 设置示例的字段。 
看下Unsafe实现的Field: 

```java 
    public static final RuntimeFieldFactory<Long> INT64 = new RuntimeFieldFactory<Long>(ID_INT64) {
        public <T> MappedSchema.Field<T> create(int number, java.lang.String name,final java.lang.reflect.Field f, IdStrategy strategy) {
            final boolean primitive = f.getType().isPrimitive();
            final long offset = us.objectFieldOffset(f);
            return new MappedSchema.Field<T>(WireFormat.FieldType.INT64, number, name, f.getAnnotation(Tag.class)) {
                public void mergeFrom(Input input, T message) throws IOException {
                    if (primitive)
                        us.putLong(message, offset, input.readInt64());
                    else
                        us.putObject(message, offset, Long.valueOf(input.readInt64()));
                }
```
```java 
 private static final sun.misc.Unsafe us = initUnsafe();
 private static sun.misc.Unsafe initUnsafe() {
    try {
        java.lang.reflect.Field f = sun.misc.Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        return (sun.misc.Unsafe) f.get(null);
    } catch (Exception e) {
    }
    return sun.misc.Unsafe.getUnsafe();
}
```

对于unsafe的操作，简直是法外之物，被reflect还凶猛，基本没法trace,出了错不好发现

