## Notes

### Templated products naming

```Cpp

template<typename T, typename T2>
class Something<T,T2> {};

template class Something<namespace1::Template1,namespace2::Template2>

```

The product name will be

```Cpp
namespace1Template1namespace2Template2Something
```
