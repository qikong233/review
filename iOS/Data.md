## 数据持久化

### NSKeyedArchiver

	归档在iOS中是一另一种形式的序列化，只要遵循了NSCoding协议的对象都可以通过它实现序列化。由于绝大多数支持存储数据的Foundation和Cocoa Touch类都准寻了NSCoding协议，因此，对于大多数类来说，归档相对而言还是比较容易实现的。

#### 1 遵循NSCoding协议

	NSCoding协议声明了两个方法，这两个方法都是必须实现的。一个是用来说明如何想对象编码到归档中，另一个说明如何进行解档来获取一个新对象。

	- 遵循协议和属性设置

	```objc
	// 1.遵循NSCoding协议
	@interface Person: NSObject<NSCoding>

	// 2.属性设置
	@property (nonatomic, strong) UIImage *avatar;
	```

	- 实现协议方法


	```objc
	//解档
	- (id)initWithCoder:(NSCoder *)aDecoder {
		if ([super init]) {
			self.avatar = [aDecoder decodeObjectForKey:@"avatar"];
		}
		return self;
	}
	//归档
	- (void)encodeWithCoder:(NSCoder *)aCoder {
		[aCoder encodeObject:self.avatar forKey:@"avatar"];
	}
	```

	- 特别注意

	如果需要归档的类是某个自定义类的子类时，就需要在归档和接档之前先实现父类的归档和解档方法。即[super encoderWithCoder:aCoder]和[super initWithCoder:aDecoder]方法

#### 2 使用
	- 需要把对象归档是调用NSKeyedArchiver的工厂方法 archiveRootObject: toFile: 方法
	```objc
	NSString *file = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject stringByAppendingPathComponent:@"person.data"];
	Person *person = [[Person alloc] init];
	person.avatar = self.avatarView.image;

	[NSKeyedArchiver archiveRootObject:person toFile:file];
	```

	- 需要从文件中解档对象就调用NSKeyedUnarchiver的一个工厂方法unarchiveObjectWithFile:即可
	
	```objc
	NSString *file = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).firstObject stringByAppendingPathComponent:@"person.data"];
	Person *person = [NSKeyedUnarchiver unarchiveObjectWithFile:file];
	if (person) {
	self.avatarView.image = pserson.avatar;
	}
	```

#### 3 注意
	
	- 必须遵循并实现NSCoding协议
	- 保存文件的拓展名可以任意制定
	- 继承时必须先调用父类的归档解档方法
