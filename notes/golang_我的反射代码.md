- [背景](#背景)
- [ToObject](#toobject)
- [FromObject](#fromobject)
- [测试程序](#测试程序)

# 背景
在早期的[gshellos](https://github.com/godevsig/gshellos)的实现中, 我们用了[tengo解释器](https://github.com/d5/tengo)来在gshell框架下解释运行`.tengo`代码.

tengo使用tengo object来表示对象, 比如int在tengo中是`tengo.Int`.

# ToObject
ToObject函数的作用是把一个原生的go对象转换为tengo对象, 支持简单的int byte bool等基础value, 以及map slice等复合value的组合.
ToObject函数是递归的, 并且设计了fast path和slow path来做value的转换.
* fast path用类型断言
* slow path用reflect

下面我把代码贴出来, 用作后面参考.
```go
// ToObject traverses the value v recursively and converts the value to tengo object.
//  Pointer values encode as the value pointed to.
//  A nil pointer/interface/slice/map encodes as the tengo.UndefinedValue value.
//  Struct values encode as tengo map. Only exported field can be encoded with filed names as map keys but with its first letter turned into lower case.
//    e.g. struct{Field1: 123, AnotherField: 456} will be converted to tengo map{field: 123, anotherField: 456}
//  int, string, float, bool, Time.time, error encodes as their corresponding tengo object.
//  slices encode as tengo Array, maps with key as string encode as tengo Map, returns ErrInvalidType if key type in map is not string.
// Returns ErrInvalidType on unsupported value type.
// Note as ToObject follows pointers, be careful with cyclic pointer references which results in infinite loop.
func ToObject(v interface{}) (tengo.Object, error) {
	// fast path
	switch v := v.(type) {
	case nil:
		return tengo.UndefinedValue, nil
	case string:
		if len(v) > tengo.MaxStringLen {
			return nil, tengo.ErrStringLimit
		}
		return &tengo.String{Value: v}, nil
	case int64:
		return &tengo.Int{Value: v}, nil
	case int:
		return &tengo.Int{Value: int64(v)}, nil
	case bool:
		if v {
			return tengo.TrueValue, nil
		}
		return tengo.FalseValue, nil
	case rune:
		return &tengo.Char{Value: v}, nil
	case byte:
		return &tengo.Char{Value: rune(v)}, nil
	case float64:
		return &tengo.Float{Value: v}, nil
	case *UserFunction:
		return v, nil
	case *tengo.UserFunction:
		return v, nil
	case tengo.Object:
		return v, nil
	case tengo.CallableFunc:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		return &tengo.UserFunction{Value: v}, nil
	case []byte:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		if len(v) > tengo.MaxBytesLen {
			return nil, tengo.ErrBytesLimit
		}
		return &tengo.Bytes{Value: v}, nil
	case error:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		return &tengo.Error{Value: &tengo.String{Value: v.Error()}}, nil
	case map[string]tengo.Object:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		return &tengo.Map{Value: v}, nil
	case map[string]int:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		kv := make(map[string]tengo.Object, len(v))
		for vk, vv := range v {
			vo, err := ToObject(vv)
			if err != nil {
				return nil, err
			}
			kv[vk] = vo
		}
		return &tengo.Map{Value: kv}, nil
	case map[string]int64:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		kv := make(map[string]tengo.Object, len(v))
		for vk, vv := range v {
			vo, err := ToObject(vv)
			if err != nil {
				return nil, err
			}
			kv[vk] = vo
		}
		return &tengo.Map{Value: kv}, nil
	case map[string]float64:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		kv := make(map[string]tengo.Object, len(v))
		for vk, vv := range v {
			vo, err := ToObject(vv)
			if err != nil {
				return nil, err
			}
			kv[vk] = vo
		}
		return &tengo.Map{Value: kv}, nil
	case map[string]string:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		kv := make(map[string]tengo.Object, len(v))
		for vk, vv := range v {
			vo, err := ToObject(vv)
			if err != nil {
				return nil, err
			}
			kv[vk] = vo
		}
		return &tengo.Map{Value: kv}, nil
	case map[string]interface{}:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		kv := make(map[string]tengo.Object, len(v))
		for vk, vv := range v {
			vo, err := ToObject(vv)
			if err != nil {
				return nil, err
			}
			kv[vk] = vo
		}
		return &tengo.Map{Value: kv}, nil
	case []tengo.Object:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		return &tengo.Array{Value: v}, nil
	case []int:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		arr := make([]tengo.Object, len(v))
		for i, e := range v {
			vo, err := ToObject(e)
			if err != nil {
				return nil, err
			}
			arr[i] = vo
		}
		return &tengo.Array{Value: arr}, nil
	case []int64:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		arr := make([]tengo.Object, len(v))
		for i, e := range v {
			vo, err := ToObject(e)
			if err != nil {
				return nil, err
			}
			arr[i] = vo
		}
		return &tengo.Array{Value: arr}, nil
	case []float64:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		arr := make([]tengo.Object, len(v))
		for i, e := range v {
			vo, err := ToObject(e)
			if err != nil {
				return nil, err
			}
			arr[i] = vo
		}
		return &tengo.Array{Value: arr}, nil
	case []string:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		arr := make([]tengo.Object, len(v))
		for i, e := range v {
			vo, err := ToObject(e)
			if err != nil {
				return nil, err
			}
			arr[i] = vo
		}
		return &tengo.Array{Value: arr}, nil
	case []interface{}:
		if v == nil {
			return tengo.UndefinedValue, nil
		}
		arr := make([]tengo.Object, len(v))
		for i, e := range v {
			vo, err := ToObject(e)
			if err != nil {
				return nil, err
			}
			arr[i] = vo
		}
		return &tengo.Array{Value: arr}, nil
	case time.Time:
		return &tengo.Time{Value: v}, nil
	}

	// slow path
	rv := reflect.ValueOf(v)
	switch rv.Kind() {
	case reflect.Ptr, reflect.Interface, reflect.Map, reflect.Slice:
		if rv.IsNil() {
			return tengo.UndefinedValue, nil
		}
	}

	rv = reflect.Indirect(rv)
	switch rv.Kind() {
	case reflect.Bool:
		if rv.Bool() {
			return tengo.TrueValue, nil
		}
		return tengo.FalseValue, nil
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return &tengo.Int{Value: rv.Int()}, nil
	case reflect.Uint, reflect.Uintptr, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
		return &tengo.Int{Value: int64(rv.Uint())}, nil
	case reflect.Float32, reflect.Float64:
		return &tengo.Float{Value: rv.Float()}, nil
	case reflect.Array, reflect.Slice:
		arr := make([]tengo.Object, rv.Len())
		for i := 0; i < rv.Len(); i++ {
			obj, err := ToObject(rv.Index(i).Interface())
			if err != nil {
				return nil, err
			}
			arr[i] = obj
		}
		return &tengo.Array{Value: arr}, nil
	case reflect.Interface:
		obj, err := ToObject(rv.Elem().Interface())
		if err != nil {
			return nil, err
		}
		return obj, nil
	case reflect.Map:
		kv := make(map[string]tengo.Object, rv.Len())
		iter := rv.MapRange()
		for iter.Next() {
			k := iter.Key()
			if k.Kind() != reflect.String {
				return nil, ErrInvalidType
			}
			v := iter.Value()
			obj, err := ToObject(v.Interface())
			if err != nil {
				return nil, err
			}
			kv[k.String()] = obj
		}
		return &tengo.Map{Value: kv}, nil
	case reflect.Struct:
		kv := make(map[string]tengo.Object, rv.NumField())
		typ := rv.Type()

		for i := 0; i < rv.NumField(); i++ {
			obj, err := ToObject(rv.Field(i).Interface())
			if err != nil {
				return nil, err
			}
			kv[firstLetterLower(typ.Field(i).Name)] = obj
		}
		return &tengo.Map{Value: kv}, nil
	}

	return nil, ErrInvalidType
}
```

# FromObject
FromObject是ToObject的反过程.
```go
// FromObject parses the tengo object and stores the result in the value pointed to by v.
// FromObject uses the inverse of the encodings that ToObject uses, allocating maps, slices, and pointers as necessary.
//
// FromObject converts tengo Map object into a struct by map look up with field names as keys.
// Filed name and tengo map key are matched in a way that the first letter is insensitive.
//   e.g. both tengo map{name: "san"} and map{Name: "san"} can be converted to struct{Name: "san"}
//  If v is nil or not a pointer, ObjectToValue returns an ErrInvalidPtr error.
//  If o is already a tengo object, it is copied to the value that v points to.
//  If v represents a *tengo.CallableFunc, and o is a tengo UserFunction object, the CallableFunc f will be copied to where v points.
// Returns ErrNotConvertibleType if o can not be converted to v, e.g. you are trying to get a map vale from tengo Array object.
// Not supported value types:
//  interface, chan, complex, func
//  In particular, interface error is not convertible.
func FromObject(v interface{}, o tengo.Object) error {
	if o == tengo.UndefinedValue {
		return nil // ignore undefined value
	}

	// fast path
	switch ptr := v.(type) {
	case *int:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToInt(o); ok {
			*ptr = v
			return nil
		}
	case *int64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToInt64(o); ok {
			*ptr = v
			return nil
		}
	case *string:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToString(o); ok {
			*ptr = v
			return nil
		}
	case *float64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToFloat64(o); ok {
			*ptr = v
			return nil
		}
	case *bool:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToBool(o); ok {
			*ptr = v
			return nil
		}
	case *rune:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToRune(o); ok {
			*ptr = v
			return nil
		}
	case *[]byte:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToByteSlice(o); ok {
			*ptr = v
			return nil
		}
	case *time.Time:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if v, ok := tengo.ToTime(o); ok {
			*ptr = v
			return nil
		}
	case *[]int:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toA := func(objArray []tengo.Object) bool {
			array := make([]int, len(objArray))
			for i, o := range objArray {
				v, ok := tengo.ToInt(o)
				if !ok {
					return false
				}
				array[i] = v
			}
			*ptr = array
			return true
		}
		switch o := o.(type) {
		case *tengo.Array:
			if toA(o.Value) {
				return nil
			}
		case *tengo.ImmutableArray:
			if toA(o.Value) {
				return nil
			}
		}
	case *[]int64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toA := func(objArray []tengo.Object) bool {
			array := make([]int64, len(objArray))
			for i, o := range objArray {
				v, ok := tengo.ToInt64(o)
				if !ok {
					return false
				}
				array[i] = v
			}
			*ptr = array
			return true
		}
		switch o := o.(type) {
		case *tengo.Array:
			if toA(o.Value) {
				return nil
			}
		case *tengo.ImmutableArray:
			if toA(o.Value) {
				return nil
			}
		}
	case *[]float64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toA := func(objArray []tengo.Object) bool {
			array := make([]float64, len(objArray))
			for i, o := range objArray {
				v, ok := tengo.ToFloat64(o)
				if !ok {
					return false
				}
				array[i] = v
			}
			*ptr = array
			return true
		}
		switch o := o.(type) {
		case *tengo.Array:
			if toA(o.Value) {
				return nil
			}
		case *tengo.ImmutableArray:
			if toA(o.Value) {
				return nil
			}
		}
	case *[]string:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toA := func(objArray []tengo.Object) bool {
			array := make([]string, len(objArray))
			for i, o := range objArray {
				v, ok := tengo.ToString(o)
				if !ok {
					return false
				}
				array[i] = v
			}
			*ptr = array
			return true
		}
		switch o := o.(type) {
		case *tengo.Array:
			if toA(o.Value) {
				return nil
			}
		case *tengo.ImmutableArray:
			if toA(o.Value) {
				return nil
			}
		}
	case *map[string]int:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toM := func(objMap map[string]tengo.Object) bool {
			mp := make(map[string]int, len(objMap))
			for k, o := range objMap {
				v, ok := tengo.ToInt(o)
				if !ok {
					return false
				}
				mp[k] = v
			}
			*ptr = mp
			return true
		}
		switch o := o.(type) {
		case *tengo.Map:
			if toM(o.Value) {
				return nil
			}
		case *tengo.ImmutableMap:
			if toM(o.Value) {
				return nil
			}
		}
	case *map[string]int64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toM := func(objMap map[string]tengo.Object) bool {
			mp := make(map[string]int64, len(objMap))
			for k, o := range objMap {
				v, ok := tengo.ToInt64(o)
				if !ok {
					return false
				}
				mp[k] = v
			}
			*ptr = mp
			return true
		}
		switch o := o.(type) {
		case *tengo.Map:
			if toM(o.Value) {
				return nil
			}
		case *tengo.ImmutableMap:
			if toM(o.Value) {
				return nil
			}
		}
	case *map[string]float64:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toM := func(objMap map[string]tengo.Object) bool {
			mp := make(map[string]float64, len(objMap))
			for k, o := range objMap {
				v, ok := tengo.ToFloat64(o)
				if !ok {
					return false
				}
				mp[k] = v
			}
			*ptr = mp
			return true
		}
		switch o := o.(type) {
		case *tengo.Map:
			if toM(o.Value) {
				return nil
			}
		case *tengo.ImmutableMap:
			if toM(o.Value) {
				return nil
			}
		}
	case *map[string]string:
		if ptr == nil {
			return ErrInvalidPtr
		}
		toM := func(objMap map[string]tengo.Object) bool {
			mp := make(map[string]string, len(objMap))
			for k, o := range objMap {
				v, ok := tengo.ToString(o)
				if !ok {
					return false
				}
				mp[k] = v
			}
			*ptr = mp
			return true
		}
		switch o := o.(type) {
		case *tengo.Map:
			if toM(o.Value) {
				return nil
			}
		case *tengo.ImmutableMap:
			if toM(o.Value) {
				return nil
			}
		}
	case *tengo.Object:
		if ptr == nil {
			return ErrInvalidPtr
		}
		*ptr = o
		return nil
	case *tengo.CallableFunc:
		if ptr == nil {
			return ErrInvalidPtr
		}
		if f, ok := o.(*tengo.UserFunction); ok {
			*ptr = f.Value
			return nil
		}
	default:
		// slow path
		rptr := reflect.ValueOf(v)
		if rptr.Kind() != reflect.Ptr || rptr.IsNil() {
			return ErrInvalidPtr
		}
		rv := rptr.Elem()
		switch rv.Kind() {
		case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
			if v, ok := tengo.ToInt64(o); ok {
				rv.SetInt(v)
				return nil
			}
		case reflect.Uint, reflect.Uintptr, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
			if v, ok := tengo.ToInt64(o); ok {
				rv.SetUint(uint64(v))
				return nil
			}
		case reflect.Float32, reflect.Float64:
			if v, ok := tengo.ToFloat64(o); ok {
				rv.SetFloat(v)
				return nil
			}
		case reflect.Ptr:
			if rv.IsNil() {
				rv.Set(reflect.New(rv.Type().Elem()))
			}
			if err := FromObject(rv.Interface(), o); err == nil {
				return nil
			}
		case reflect.Array, reflect.Slice:
			toA := func(objArray []tengo.Object) bool {
				array := reflect.MakeSlice(rv.Type(), len(objArray), len(objArray))
				for i, o := range objArray {
					if o == tengo.UndefinedValue {
						continue
					}
					elem := array.Index(i)
					if err := FromObject(elem.Addr().Interface(), o); err != nil {
						return false
					}
				}
				rv.Set(array)
				return true
			}

			switch o := o.(type) {
			case *tengo.Array:
				if toA(o.Value) {
					return nil
				}
			case *tengo.ImmutableArray:
				if toA(o.Value) {
					return nil
				}
			}
		case reflect.Map:
			toM := func(objMap map[string]tengo.Object) bool {
				typ := rv.Type()
				if typ.Key().Kind() != reflect.String {
					return false
				}
				mp := reflect.MakeMapWithSize(typ, len(objMap))
				elemPtr := reflect.New(typ.Elem())
				for k, o := range objMap {
					if o == tengo.UndefinedValue {
						continue
					}
					if err := FromObject(elemPtr.Interface(), o); err != nil {
						return false
					}
					mp.SetMapIndex(reflect.ValueOf(k), elemPtr.Elem())
				}
				rv.Set(mp)
				return true
			}

			switch o := o.(type) {
			case *tengo.Map:
				if toM(o.Value) {
					return nil
				}
			case *tengo.ImmutableMap:
				if toM(o.Value) {
					return nil
				}
			}
		case reflect.Struct:
			toStruct := func(objMap map[string]tengo.Object) bool {
				typ := rv.Type()
				for i := 0; i < rv.NumField(); i++ {
					fieldName := typ.Field(i).Name
					obj, ok := objMap[firstLetterLower(fieldName)]
					if !ok {
						obj, ok = objMap[fieldName]
					}
					if ok {
						if obj == tengo.UndefinedValue {
							continue
						}
						field := rv.Field(i)
						if err := FromObject(field.Addr().Interface(), obj); err != nil {
							return false
						}
					}
				}
				return true
			}
			switch o := o.(type) {
			case *tengo.Map:
				if toStruct(o.Value) {
					return nil
				}
			case *tengo.ImmutableMap:
				if toStruct(o.Value) {
					return nil
				}
			}
		}
	}
	return errNotConvertible(reflect.ValueOf(v).Elem().String(), o)
}
```

# 测试程序
```go
package gshellos

import (
	"errors"
	"fmt"
	"reflect"
	"regexp"
	"strings"
	"testing"
	"time"

	"github.com/d5/tengo/v2"
)

func TestToObject(t *testing.T) {
	testb := true
	var testi uint32 = 88
	testf := 33.33
	var itf interface{}
	itf = testf

	type student struct {
		Name    string
		Age     int
		Scores  map[string]float32
		Friends []student
	}

	empty := struct {
		A string
		B int
		C float64
		D []int
		E []int64
		F []float64
		G []tengo.Object
		H []string
		I []byte
		J []interface{}
		K tengo.CallableFunc
		L error
		M map[string]int
		N map[string]int64
		O map[string]float64
		P map[string]string
		Q map[string]interface{}
		R map[string]tengo.Object
		S *int
	}{}

	cases := []struct {
		v    interface{}
		want string
	}{
		{&tengo.Int{Value: 123}, `&tengo.Int{ObjectImpl:tengo.ObjectImpl{}, Value:123}`},
		{nil, `&tengo.Undefined{ObjectImpl:tengo.ObjectImpl{}}`},
		{1, `&tengo.Int{ObjectImpl:tengo.ObjectImpl{}, Value:1}`},
		{"hello world", `&tengo.String{ObjectImpl:tengo.ObjectImpl{}, Value:"hello world", runeStr:[]int32(nil)}`},
		{99.99, `&tengo.Float{ObjectImpl:tengo.ObjectImpl{}, Value:99.99}`},
		{false, `&tengo.Bool{ObjectImpl:tengo.ObjectImpl{}, value:false}`},
		{'@', `&tengo.Char{ObjectImpl:tengo.ObjectImpl{}, Value:64}`},
		{byte(56), `&tengo.Char{ObjectImpl:tengo.ObjectImpl{}, Value:56}`},
		{[]byte("567"), `&tengo.Bytes{ObjectImpl:tengo.ObjectImpl{}, Value:[]uint8{0x35, 0x36, 0x37}}`},
		{errors.New("err"), `&tengo.Error{ObjectImpl:tengo.ObjectImpl{}, Value:\(\*tengo.String\)\(0x[0-9a-f]+\)}`},
		{map[string]int{"zhangsan": 30, "lisi": 35},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((zhangsan)|(lisi))":\(\*tengo.Int\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{map[string]int64{"zhangsan": 30, "lisi": 35},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((zhangsan)|(lisi))":\(\*tengo.Int\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{map[string]string{"zhangsan": "teacher", "lisi": "student"},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((zhangsan)|(lisi))":\(\*tengo.String\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{map[string]interface{}{"zhangsan": 30, "lisi": "student"},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((zhangsan)|(lisi))":\(\*tengo.((String)|(Int))\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{map[string]float64{"zhangsan": 30.1, "lisi": 35.2},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((zhangsan)|(lisi))":\(\*tengo.Float\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{[2]int{11, 13},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.Int\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{[]int{101, 103, 105},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.Int\)\(0x[0-9a-f]+\),? ?){3}}}$`},
		{[]int64{101, 103, 105},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.Int\)\(0x[0-9a-f]+\),? ?){3}}}$`},
		{[]float64{101.1, 103.1, 105.1},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.Float\)\(0x[0-9a-f]+\),? ?){3}}}$`},
		{[]string{"ni", "hao", "ma"},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.String\)\(0x[0-9a-f]+\),? ?){3}}}$`},
		{[]interface{}{"ni", "hao", 123},
			`^&tengo.Array{ObjectImpl:tengo.ObjectImpl{}, Value:\[\]tengo.Object{(\(\*tengo.((String)|(Int))\)\(0x[0-9a-f]+\),? ?){3}}}$`},
		{time.Now(), `^&tengo.Time{ObjectImpl:tengo.ObjectImpl{}, Value:time.Time{.*}}$`},
		{&testb, `&tengo.Bool{ObjectImpl:tengo.ObjectImpl{}, value:true}`},
		{int16(55), `&tengo.Int{ObjectImpl:tengo.ObjectImpl{}, Value:55}`},
		{&testi, `&tengo.Int{ObjectImpl:tengo.ObjectImpl{}, Value:88}`},
		{&testf, `&tengo.Float{ObjectImpl:tengo.ObjectImpl{}, Value:33.33}`},
		{itf, `&tengo.Float{ObjectImpl:tengo.ObjectImpl{}, Value:33.33}`},
		{student{"lisi", 20, map[string]float32{"yuwen": 86.5, "shuxue": 83.1}, []student{{Name: "zhangsan"}, {Name: "wangwu"}}},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((age)|(friends)|(name)|(scores))":\(\*tengo.((Int)|Array|String|Map)\)\(0x[0-9a-f]+\),? ?){4}}}$`},
		{map[string]student{"zhangsan": {Name: "zhangsan"}, "lisi": {Name: "lisi"}},
			`^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{("((lisi)|(zhangsan))":\(\*tengo.Map\)\(0x[0-9a-f]+\),? ?){2}}}$`},
		{empty, `^&tengo.Map{ObjectImpl:tengo.ObjectImpl{}, Value:map\[string\]tengo.Object{"a":\(\*tengo.String\)\(0x[0-9a-f]+\), "b":\(\*tengo.Int\)\(0x[0-9a-f]+\), "c":\(\*tengo.Float\)\(0x[0-9a-f]+\), ("[d-s]{1}":\(\*tengo.Undefined\)\(0x[0-9a-f]+\),? ?){16}}}$`},
	}

	for _, c := range cases {
		if len(c.want) == 0 {
			t.Error("empty want")
		}
		obj, err := ToObject(c.v)
		if err != nil {
			t.Error(err)
			continue
		}
		got := fmt.Sprintf("%#v", obj)
		//t.Logf("%v\n", obj)
		if got == c.want {
			continue
		}
		re := regexp.MustCompile(c.want)
		if !re.MatchString(got) {
			t.Errorf("want: %s, got: %s", c.want, got)
		}
	}

	_, err := ToObject([]complex64{complex(1, -2), complex(1.0, -1.4)})
	if err != ErrInvalidType {
		t.Error("complex supported?")
	}

	_, err = ToObject(map[string]interface{}{"a": complex(1, -2), "b": complex(1.0, -1.4)})
	if err != ErrInvalidType {
		t.Error("complex supported?")
	}
}

func TestFromObject(t *testing.T) {
	obj, _ := ToObject(55)
	emptyCases := []interface{}{
		(*int)(nil),
		(*int64)(nil),
		(*string)(nil),
		(*float64)(nil),
		(*bool)(nil),
		(*rune)(nil),
		(*[]byte)(nil),
		(*time.Time)(nil),
		(*[]int)(nil),
		(*[]int64)(nil),
		(*[]float64)(nil),
		(*[]string)(nil),
		(*map[string]int)(nil),
		(*map[string]int64)(nil),
		(*map[string]float64)(nil),
		(*map[string]string)(nil),
		(*tengo.Object)(nil),
		(*tengo.CallableFunc)(nil),
		(*int32)(nil),
		nil,
	}

	for _, c := range emptyCases {
		err := FromObject(c, obj)
		if err != ErrInvalidPtr {
			t.Fatal("empty ptr error expected")
		}
	}

	var got tengo.Object
	err := FromObject(&got, obj)
	if err != nil {
		t.Error(err)
	}
	if !reflect.DeepEqual(got, obj) {
		t.Errorf("want: %#v, got: %#v", obj, got)
	}

	testf := func(args ...tengo.Object) (tengo.Object, error) {
		return nil, nil
	}
	var gotf func(args ...tengo.Object) (tengo.Object, error)
	fobj, err := ToObject(testf)
	if err != nil {
		t.Error(err)
	}
	err = FromObject(&gotf, fobj)
	if err != nil {
		t.Error(err)
	}
	var itf interface{} = gotf
	gotstring := fmt.Sprintf("%#v", itf)
	wantstring := `(func(...tengo.Object) (tengo.Object, error))`
	if !strings.Contains(gotstring, wantstring) {
		t.Errorf("want: %s, got: %s", wantstring, gotstring)
	}
	err = FromObject(&gotf, obj)
	if !errors.As(err, &ErrNotConvertibleType{}) {
		t.Error(err)
	}

	type student struct {
		Name       string
		Age        int
		Scores     map[string]float32
		Classmates []student
		Deskmate   *student
		Friends    map[string]*student
	}

	studentA := student{
		"lisi",
		20,
		map[string]float32{"yuwen": 86.5, "shuxue": 83.1},
		[]student{{Name: "zhangsan"}, {Name: "wangwu"}},
		nil,
		nil,
	}

	studentB := student{
		"zhangsan",
		21,
		map[string]float32{"yuwen": 78.5, "shuxue": 96.1},
		[]student{{Name: "lisi"}, {Name: "wangwu"}},
		&studentA,
		map[string]*student{"si": &studentA},
	}

	cases := []interface{}{
		"hello world",
		55,
		int64(33),
		55.77,
		true,
		'U',
		[]byte{1, 2, 3, 4, 5},
		time.Now(),
		[]int{22, 33, 44},
		[]int64{22, 33, 44},
		[]float64{22.1, 33.2, 44.9},
		[]string{"ni", "hao", "ma"},
		map[string]int{"A": 1, "b": 15},
		map[string]int64{"A": 1, "b": 15},
		map[string]float64{"a": 1.54, "U": 3.14},
		map[string]string{"a": "12345", "U": "hello world"},
		int16(12),
		uint16(12),
		float32(1.2345),
		studentB,
		studentA,
	}

	for _, c := range cases {
		t.Logf("c: %#v", c)
		obj, err := ToObject(c)
		if err != nil {
			t.Fatal(err)
		}
		t.Logf("obj: %#v", obj)

		ptr := reflect.New(reflect.TypeOf(c))
		err = FromObject(ptr.Interface(), obj)
		if err != nil {
			t.Error(err)
			continue
		}
		v := ptr.Elem().Interface()
		t.Logf("v: %#v", v)
		if !reflect.DeepEqual(c, v) {
			t.Errorf("want: %#v, got: %#v", c, v)
		}
	}
	//t.Error("err")
}
```
