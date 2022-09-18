---
title: "Design Pattern With Go: Factory"
date: 2021-03-25T15:06:14+08:00
tags:
  - golang
  - design-pattern
categories:
  - golang
  - design-pattern
---
这次介绍的设计模式是工厂模式，这是一个比较常见的创建型模式。一般情况下，工厂模式分为三种：简单工厂、工厂方法和抽象工厂，下面慢慢举例介绍下。

# 简单工厂
考虑一个加密程序的应用场景，一个加密程序可能提供了AES，DES等加密方法，这些加密方式都实现了同一个接口ICipher，它有两个方法分别是 Encript 和 Decript。我们使用加密程序的时候会希望简单的指定加密方式，然后传入原始数据以及必要参数，然后就能得到想要的加密数据。这个功能用简单工厂如何实现呢？

## 模式结构

简单工厂模式包含一下几个角色：

- Factory（工厂角色），负责创建所有实例。
- Product（抽象产品角色），指工厂所创建的实例的基类，在 golang 中通常为接口。
- ConcreteProduce（具体产品），指工厂所创建的具体实例的类型。

在这个加密程序的例子中，工厂角色的职责是返回加密函数；抽象产品角色是所有加密类的基类，在 golang 中是定义了加密类通用方法的接口；具体产品是指具体的加密类，如 AES、DES 等等。我们可以用 UML 关系图来表示这几个角色之间的关系：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20210329101034.png)

## 代码设计
依据 UML 关系图，我们可以设计出采用简单工厂模式的加密代码。首先是 ICipher 接口，定义了 Encript 和 Decript 两个方法：
``` golang
type ICipher interface {
	Encrypt([]byte) ([]byte, error)
	Decrypt([]byte) ([]byte, error)
}
```
然后根据这个接口，分别实现 AESCipher 和 DESCipher 两个加密类。

**AESCipher**:
``` golang
type AESCipher struct {
}

func NewAESCipher() *AESCipher {
	return &AESCipher{}
}

func (c AESCipher) Encrypt(data []byte) ([]byte, error) {
	return nil, nil
}

func (c AESCipher) Decrypt(data []byte) ([]byte, error) {
	return nil, nil
}

```

**DESCipher**:
```golang
type DESCipher struct {
}

func NewDesCipher() *DESCipher {
	return &DESCipher{}
}

func (c DESCipher) Encrypt(data []byte) ([]byte, error) {
	return nil, nil
}

func (c DESCipher) Decrypt(data []byte) ([]byte, error) {
	return nil, nil
}

```
最后是一个工厂角色，根据传入的参数返回对应的加密类，Java 需要实现一个工厂类，这里我们用一个函数来做加密类工厂：
```golang
func CipherFactory(cType string) ICipher {
	switch cType {
	case "AES":
		return NewAESCipher()
	case "DES":
		return NewDesCipher()
	default:
		return nil
	}
}
```
这样，通过调用 CipherFactory 传入所需的加密类型，就可以得到所需要的加密类实例了。
```golang
func TestCipherFactory(t *testing.T) {
	c := CipherFactory("RSA")
	if c != nil {
		t.Fatalf("unsupport RSA")
	}

	c = CipherFactory("AES")
	if reflect.TypeOf(c) != reflect.TypeOf(&AESCipher{}) {
		t.Fatalf("cipher type should be AES")
	}

	c = CipherFactory("DES")
	if reflect.TypeOf(c) != reflect.TypeOf(&DESCipher{}) {
		t.Fatalf("cipher type should be DES")
	}
}
```

## 小结
简单工厂将业务代码和创建实例的代码分离，使职责更加单一。不过，它将所有创建实例的代码都放到了 CipherFactory 中，当加密类增加的时候会增加工厂函数的复杂度，产品类型增加时需要更新工厂函数这一操作也是违反了“开闭原则”，所以简单工厂更适合负责创建的对象比较少的场景。

# 工厂方法
为了让代码更加符合“开闭原则”，我们可以给每个产品都增加一个工厂子类，每个子类生成具体的产品实例，将工厂方法化，也就是现在要介绍的工厂方法模式。

## 模式结构
工厂方法和和简单工厂相比，将工厂角色细分成抽象工厂和具体工厂：
- Product（抽象产品）：定义产品的接口。
- ConcreteFactory（具体产品）：具体的产品实例。
- Factory（抽象工厂）：定义工厂的接口。
- ConcreteFactory（具体工厂）：实现抽象工厂，生产具体产品。

可以使用如下的 UML 图来表示这几个角色直接的关系：
![](https://raw.githubusercontent.com/LooJee/medias/master/images/20210329142446.png)

## 代码设计
抽象产品角色和具体产品角色就不再定义了，和简单工厂相同，具体展示一下抽象工厂角色和具体工厂角色。
抽象工厂角色定义了一个方法，用于创建对应的产品：
```golang
type ICipherFactory interface {
	GetCipher() ICipher
}
```
根据这个接口，分别定义出 AESCipherFactory、和 DESCipherFactory 两个子类工厂。

**AESCipherFactory**
```golang
type AESCipherFactory struct {
}

func (AESCipherFactory) GetCipher() ICipher {
	return NewAESCipher()
}

func NewAESCipherFactory() *AESCipherFactory {
	return &AESCipherFactory{}
}
```

**DESCipherFactory**
```golang
type DESCipherFactory struct {
}

func (DESCipherFactory) GetCipher() ICipher {
	return NewDESCipher()
}

func NewDESCipherFactory() *DESCipherFactory {
	return &DESCipherFactory{}
}
```
然后编写一个单元测试来检验我们的代码：
```golang
func TestCipherFactory(t *testing.T) {
	var f ICipherFactory = NewAESCipherFactory()
	if reflect.TypeOf(f.GetCipher()) != reflect.TypeOf(&AESCipher{}) {
		t.Fatalf("should be AESCipher")
	}

	f = NewDESCipherFactory()
	if reflect.TypeOf(f.GetCipher()) != reflect.TypeOf(&DESCipher{}) {
		t.Fatalf("should be DESCipher")
	}
}
```

## 小结
在工厂方法模式中，定义了一个工厂接口，然后根据各个产品子类定义实现这个接口的子类工厂，通过子类工厂来返回产品实例。这样修改创建实例代码只需要修改子类工厂，新增实例时只需要新增具体工厂和具体产品，而不需要修改其它代码，符合“开闭原则”。不过，当具体产品较多的时候，系统中类的数量也会成倍的增加，一定程度上增加了系统的复杂度。而且，在实际使用场景中，可能还需要使用反射等技术，增加了代码的抽象性和理解难度。

# 抽象工厂
下面再用加密这个例子可能不太好，不过我们假设需求都合理吧。现在需求更加细化了，分别需要 64 位 key 和 128 位 key 的 AES 加密库以及 64 位 key 和 128 位 key 的 DES 加密库。如果使用工厂方法模式，我们一共需要定义 4 个具体工厂和 4 个具体产品。
```golang
AESCipher64
AESCipher128
AESCipherFactory64
AESCipherFactory128
DESCipher64
DESCipher128
DESCipherFactory64
DESCipherFactory128
```
这时候，我们可以把有关联性的具体产品组合成一个产品组，例如AESCipher64 和 AESCipher128，让它们通过同一个工厂 AESCipherFactory 来生产，这样就可以简化成 2 个具体工厂和 4 个具体产品
```golang
AESCipher64
AESCipher128
AESCipherFactory
DESCipher64
DESCipher128
DESCipherFactory
```
这就是抽象工厂模式。

## 模式结构
抽象工厂共有 4 个角色：
- AbstractFactory（抽象工厂）：定义工厂的接口。
- ConcreteFactory（具体工厂）：实现抽象工厂，生产具体产品。
- AbstractProduct（抽象产品）：定义产品的接口。
- Product（具体产品）：具体的产品实例。

根据角色定义我们可以画出抽象工厂的 UML 关系图：
![](https://raw.githubusercontent.com/LooJee/medias/master/images/20210329162259.png)

## 代码设计
抽象产品和具体产品的定义与工厂方法类似：

**抽象产品**：
```golang
type ICipher interface {
	Encrypt(data, key[]byte) ([]byte, error)
	Decrypt(data, key[]byte) ([]byte, error)
}
```
**AESCipher64**：
```golang
type AESCipher64 struct {
}

func NewAESCipher64() *AESCipher64 {
	return &AESCipher64{}
}

func (AESCipher64) Encrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}

func (AESCipher64) Decrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}
```
**AESCipher128**：
``` golang

type AESCipher128 struct {
}

func NewAESCipher128() *AESCipher128 {
	return &AESCipher128{}
}

func (AESCipher128) Encrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}

func (AESCipher128) Decrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}
```
**AESCipher128**：
```golang
type c struct {
}

func NewDESCipher64() *DESCipher64 {
	return &DESCipher64{}
}

func (DESCipher64) Encrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}

func (DESCipher64) Decrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}
```
**DESCipher128**：
```golang
type DESCipher128 struct {
}

func NewDESCipher128() *DESCipher128 {
	return &DESCipher128{}
}

func (DESCipher128) Encrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}

func (DESCipher128) Decrypt(data, key []byte) ([]byte, error) {
	return nil, nil
}

```
抽象工厂角色和工厂方法相比需要增加 GetCipher64 和 GetCipher128 两个方法定义：
```golang
type ICipherFactory interface {
	GetCipher64() ICipher
	GetCipher128() ICipher
}
```
然后分别实现 AESCipherFactory 和 DesCipherFactory 两个具体工厂：

**AESCipherFactory**：
```golang
type AESCipherFactory struct {
}

func (AESCipherFactory) GetCipher64() ICipher {
	return NewAESCipher64()
}

func (AESCipherFactory) GetCipher128() ICipher {
	return NewAESCipher128()
}

func NewAESCipherFactory() *AESCipherFactory {
	return &AESCipherFactory{}
}
```
**DESCipherFactory**：
```golang
type DESCipherFactory struct {
}

func (DESCipherFactory) GetCipher64() ICipher {
	return NewDESCipher64()
}

func (DESCipherFactory) GetCipher128() ICipher {
	return NewDESCipher128()
}

func NewDESCipherFactory() *DESCipherFactory {
	return &DESCipherFactory{}
}
```
编写单元测试验证我们的代码：
```golang
func TestAbstractFactory(t *testing.T) {
	var f = NewCipherFactory("AES")
	if reflect.TypeOf(f.GetCipher64()) != reflect.TypeOf(&AESCipher64{}) {
		t.Fatalf("should be AESCipher64")
	}

	if reflect.TypeOf(f.GetCipher128()) != reflect.TypeOf(&AESCipher128{}) {
		t.Fatalf("should be AESCipher128")
	}

	f = NewCipherFactory("DES")
	if reflect.TypeOf(f.GetCipher64()) != reflect.TypeOf(&DESCipher64{}) {
		t.Fatalf("should be DESCipher64")
	}

	if reflect.TypeOf(f.GetCipher128()) != reflect.TypeOf(&DESCipher128{}) {
		t.Fatalf("should be DESCipher128")
	}
}
```

## 小结
抽象工厂模式也符合单一职责原则和开闭原则，不过需要引入大量的类和接口，使代码更加复杂。并且，当增加新的具体产品时，需要修改抽象工厂和所有的具体工厂。

# 总结
今天介绍了创建型模式之工厂模式，工厂模式包括简单工厂、工厂方法和抽象工厂。简单工厂的复杂性比较低，但是不像工厂方法和抽象工厂符合单一职责原则和开闭原则。实际使用时，通常会选择符合开闭原则，复杂度也不是特别高的工厂方法。如果有特别需求可以选择使用抽象工厂。