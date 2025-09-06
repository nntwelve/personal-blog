Đây là bài viết nằm trong Series NestJS thực chiến, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

# Đặt vấn đề 📜
Xin chào mọi người, ở bài viết trước chúng ta đã tìm hiểu về State Design Pattern, hôm nay chúng ta sẽ cùng nối tiếp bài viết với một Design Pattern không kém phần hấp dẫn chính là Builder 👷🏗️🏢.  Như thường lệ chúng ta sẽ nói qua về khai niệm sau đó bắt đầu với ví dụ về code OOP cơ bản và cuối cùng là apply vô dự án NestJS thực tế nhé 😍.
# Lý thuyết 📃
> Disclaimer: bài viết này dựa theo kiến thức của mình nên nếu có sai sót mọi người góp ý giúp mình nhé.

## Khái niệm & Vấn đề ⚠️
> Builder is a creational design pattern that lets you construct complex objects step by step. The pattern allows you to produce different types and representations of an object using the same construction code.
> 
> Builder dùng để tạo ra những object phức tạp từng bước một giúp chung ta tạo ra những kiểu object khác nhau nhưng có cùng cách khởi tạo.

Trong thực tế đôi lúc chúng ta sẽ hay bắt gặp trường hợp các object có mối quan hệ với nhau và việc triển khai chúng thành các class khá là khó khăn, chúng ta sẽ cùng xét ví dụ sau:

Giả sử chúng ta đang phát triển 1 game nhập vai với nhiều class nhân vật khác nhau, mỗi nhân vật sẽ sử dụng các trang bị chung như vũ khí chính, áo giáp, giầy nhưng sẽ có các trang bị riêng tuỳ theo class nhân vật:
- Kiếm sĩ: không dùng khiên, không có linh thú, đội mũ
- Đấu sĩ: dùng khiên, không có linh thú, đội mũ
- Pháp sư: không dùng khiên, không có linh thú, đội mũ, 
- Thầy tu: không dùng khiên, không có linh thú, không đội mũ
- Triệu hồi sư: không dùng khiên, có linh thú, có đội mũ

Chúng ta sẽ triển khai vào OOP như thế nào cho các class nhân vật trên? 

**Cách 1**:
Dùng class chung và các class còn lại sẽ kế thừa:
```typescript
class Character {
	constructor(main_weapon: string, armor: string, greaves: string) {...}
}
class SwordMan extends Character {
	constructor(main_weapon: string, armor: string, greaves: string, hat: string) {...}
}
class Gladiator extends Character {
	constructor(main_weapon: string, armor: string, greaves: string, sub_weapon: string, hat: string) {...}
}
class Magician extends Character {
	constructor(main_weapon: string, armor: string, greaves: string, hat: string) {...}
}
class Monk extends Character {
	constructor(main_weapon: string, armor: string, greaves: string) {...}
}
class Summoner extends Character {
	constructor(main_weapon: string, armor: string, greaves: string, hat: string, beast: string) {...}
}
```
Cách này ổn nếu như chúng ta chỉ có bao nhiêu đây class và trang bị, nhưng nếu chúng ta có nhiều class nhân vật và trang bị thì vấn đề sẽ phát sinh. Ví dụ chúng ta có 10 class nhân vật và thêm mới 1 loại trang bị là dây chuyền ma pháp và trang bị này áp dụng cho 8/10 class, việc chúng ta làm là phải vào 8 class đó để bổ sung thêm field 😦.

**Cách 2**: Thay vì tách ra các subclass chúng ta sẽ chỉ dùng chung 1 class duy nhất là Character và việc khởi tạo chỉ cần new Character và những trang bị nào không hỗ trợ sẽ để null. 
```typescript
class Character {
	constructor(type: string, main_weapon: string, armor: string, greaves: string, sub_weapon: string, hat: string, beast: string) {...}
}
const sword_man = new Character('sword_man', ‘main_weapon’, ‘armor’, ‘greaves’, null, ‘hat’, null);
const gladiator = new Character(‘gladiator’, ‘main_weapon’, ‘armor’, ‘greaves’, ‘’sub_weapon’, ‘hat’, null);
const magician = new Character(‘magician’, ‘main_weapon’, ‘armor’, ‘greaves’, null, ‘hat’, null);
const monk = new Character(‘monk’, ‘main_weapon’, ‘armor’, ‘greaves’, null, null, null);
const summoner = new Character(‘summoner’, ‘main_weapon’, ‘armor’, ‘greaves’, null, ‘hat’, ‘beast’);
```
Cách này cũng không thật sự giải quyết được vấn đề gặp ở cách 1, mọi người thấy việc khai báo null hàng loạt như thế nhìn sẽ khó chịu hơn nữa đúng không.

Để xử lý vấn đề này một cách rõ ràng và mạch lạc chúng ta cần đến **Builder Design Pattern**. Có thể tóm gọn nội dung lý thuyết như sau:
- Builder sẽ chia quá trình ra làm nhiều bước: như ở trên mỗi bước sẽ build 1 item
- Một số bước được chia ở trên có thể không thực hiện: ví dụ như khi tạo ra `sword_man` chúng ta không cần `sub_weapon`, `beast`.
- Mỗi bước có thể có cách triển khai khác nhau: tuỳ vào người tạo ra nhân vật mà có thể có cách build ra kết quả khác nhau (chúng ta sẽ bàn sâu hơn ở ví dụ thứ 2).
- Có thể uỷ nhiệm cho class Director thực hiện việc khởi tạo thay vì trực tiếp khởi tạo

![image.png](https://images.viblo.asia/d7405d03-632e-4426-9ea4-995d3dd527fe.png)

Nguồn: https://refactoring.guru

## Solution 🫡
Chúng ta sẽ cùng nhau giải quyết vấn đề trên, bắt đầu bằng việc tạo ra *Builder Interface* `CharacterEquipment`. Thay vì gôm toàn bộ vào constructor builder sẽ chia ra thành các method tương ứng với từng bước.
```typescript
class Equipment {
  main_weapon: string;
  armor: string;
  greaves: string;
  sub_weapon: string;
  hat: string;
  beast: string;
  necklace: string;
}

interface CharacterEquipment {
  buildMainWeapon(item: string): void;
  buildArmor(item: string): void;
  buildGreaves(item: string): void;
  buildSubWeapon(item: string): void;
  buildHat(item: string): void;
  buildBeast(item: string): void;
  buildNecklace(item: string): void;
  getEquipment(): Equipment;
  reset(): void;
}
```
Sau đó chúng ta sẽ tạo ra các *Builder Concrete Implement* để triển khai logic cho các bước trên:
```typescript
class SwordmanBuilder implements CharacterEquipment {
  private equipment = new Equipment();
  buildNecklace(item: string): void {
    this.equipment.necklace = item;
  }
  getEquipment(): Equipment {
    return this.equipment;
  }
  reset(): void {
    this.equipment = new Equipment();
  }
  buildMainWeapon(item: string): void {
    this.equipment.main_weapon = item;
  }
  buildArmor(item: string): void {
    this.equipment.armor = item;
  }
  buildGreaves(item: string): void {
    this.equipment.greaves = item;
  }
  buildSubWeapon(item: string): void {
    throw new Error('Method not implemented.');
  }
  buildHat(item: string): void {
    this.equipment.hat = item;
  }
  buildBeast(item: string): void {
    throw new Error('Method not implemented.');
  }
}
```
Tới đây chúng ta có 2 hướng: **sử dụng trực tiếp** hoặc **tiếp tục với Director**. Mình sẽ triển khai cả 2 để mọi người có cái nhìn tổng quan. Nếu không có Director chúng ta sẽ sử dụng như sau:
```typescript
// Usage without director
const sword_man_builder = new SwordmanBuilder();
sword_man_builder.buildMainWeapon('Sword');
sword_man_builder.buildArmor('Plate Armor');
sword_man_builder.buildGreaves('Steel Greaves');
sword_man_builder.buildHat('Warrior Helmet');
sword_man_builder.buildNecklace('Magic Necklace');

const swordmanEquipment = sword_man_builder.getEquipment();
console.log({ swordmanEquipment });
```
Chúng ta sẽ thực hiện việc khởi tạo builder, sau đó trực tiếp chỉ đạo builder build những item. Các bạn có thể thử chạy đoạn code trên để xem kết quả:

![image.png](https://images.viblo.asia/4b042356-0502-4d58-9f36-b42c3e8a29fd.png)

Đối với trường hợp dùng Director mọi người hình dung thay vì chúng ta phải tự mình chỉ đạo builder thì chúng ta thuê 1 ông chủ thầu để ông ấy xử lý công việc thay chúng ta, chúng ta chỉ cần đưa ra yêu cầu và nhận về kết quả.
```typescript
class EquipmentDirector {
  private builder: CharacterEquipment;

  constructor(builder: CharacterEquipment) {
    this.builder = builder;
  }

  buildEquipment(type: string) {
    switch (type) {
      case 'Swordman':
        this.builder.buildMainWeapon('Sword');
        this.builder.buildArmor('Plate Armor');
        this.builder.buildGreaves('Steel Greaves');
        this.builder.buildHat('Warrior Helmet');
        this.builder.buildNecklace('Magic Necklace');
        break;
      case 'Monk':
        this.builder.buildMainWeapon('Staff');
        this.builder.buildArmor('Leather Armor');
        this.builder.buildGreaves('Cloth Greaves');
        break;
      case 'Gladiator':
        this.builder.buildMainWeapon('Spear');
        this.builder.buildArmor('Chain Armor');
        this.builder.buildGreaves('Leather Greaves');
        this.builder.buildSubWeapon('Shield');
        this.builder.buildHat('Gladiator Helmet');
        this.builder.buildNecklace('Magic Necklace');
        break;
      case 'Summoner':
        this.builder.buildMainWeapon('Magic Wand');
        this.builder.buildArmor('Robe');
        this.builder.buildGreaves('Cloth Greaves');
        this.builder.buildHat('Summoner Hat');
        this.builder.buildBeast('Phoenix');
        this.builder.buildNecklace('Magic Necklace');
        break;
      default:
        break;
    }
    return this.builder.getEquipment();
  }
  
  changeBuilder(builder: CharacterEquipment) { // Có thể thay đổi builder
    this.builder = builder;
  }

  resetBuilder() {
    this.builder.reset();
  }
}
```
Có thể thấy ở method `buildEquipment` Director biết rõ được cách để build ra kết quả chúng ta cần sẽ đi qua những bước nào. Việc của chúng ta bây giờ sẽ đơn giản hơn rất nhiều:
```typescript
// Usage with director
const sword_man_builder = new SwordmanBuilder();
const equipmentDirector = new EquipmentDirector(sword_man_builder);
const swordmanEquipment = equipmentDirector.buildEquipment('Swordman');
console.log({ swordmanEquipment });

const monk_builder = new MonkBuilder();
equipmentDirector.changeBuilder(monk_builder);
const monkEquipment = equipmentDirector.buildEquipment('Monk');
console.log({ monkEquipment });
```
Giờ chúng ta không cần bận tâm đến việc phải yêu cầu builder làm những bước nào, việc nào khó có Director lo 😁. 

Nếu mọi người để ý thì có thể thấy ở ví dụ trên trong các method của Builder chúng ta chỉ gán giá trị nhận vào từ param hoặc trả về lỗi nếu method không hỗ trợ.  Chúng ta sẽ cùng đi qua thêm một ví dụ nữa để làm rõ hơn về cách khía cạnh của Builder Design Pattern.

Lấy ví dụ chúng ta có đội ngũ Admin/Tester cần tiến hành test dự án trước khi ra mắt, họ sẽ cần tạo ra bộ item có chỉ số tối đa để trang bị cho nhân vật. 
```typescript
class Equipment {
  main_weapon: {
    name: string;
    attack: number;
  };
  armor: {
    name: string;
    defense: number;
  };
  greaves: {
    name: string;
    agility: number;
  };
  sub_weapon: {
    name: string;
    defense: number;
  };
}

interface EquipmentCraft {
  craftMainWeapon(material: string): void;
  craftArmor(material: string): void;
  craftGreaves(material: string): void;
  craftSubWeapon(material: string): void;
  getEquipment(): Equipment;
  reset(): void;
}
```
Ở ví dụ này chúng ta sẽ thêm chỉ số cho các item. Tên method trong Builder Interface tương ứng với việc tạo ra các item khác nhau.
```typescript
class UserBuilder implements EquipmentCraft {
  private equipment = new Equipment();
  getEquipment(): Equipment {
    return this.equipment;
  }
  reset(): void {
    this.equipment = new Equipment();
  }
  craftMainWeapon(material: string): void {
    this.equipment.main_weapon = {
      name: material,
      attack: Math.floor(Math.random() * 100),
    };
  }
  craftArmor(material: string): void {
    this.equipment.armor = {
      name: material,
      defense: Math.floor(Math.random() * 100),
    };
  }
  craftGreaves(material: string): void {
    this.equipment.greaves = {
      name: material,
      agility: Math.floor(Math.random() * 100),
    };
  }
  craftSubWeapon(material: string): void {
    this.equipment.sub_weapon = {
      name: material,
      defense: Math.floor(Math.random() * 100),
    };
  }
}

class TesterBuilder implements EquipmentCraft {
  private equipment = new Equipment();
  getEquipment(): Equipment {
    return this.equipment;
  }
  reset(): void {
    this.equipment = new Equipment();
  }
  craftMainWeapon(material: string): void {
    this.equipment.main_weapon = {
      name: material,
      attack: 100,
    };
  }
  craftArmor(material: string): void {
    this.equipment.armor = {
      name: material,
      defense: 100,
    };
  }
  craftGreaves(material: string): void {
    this.equipment.greaves = {
      name: material,
      agility: 100,
    };
  }
  craftSubWeapon(material: string): void {
    this.equipment.sub_weapon = {
      name: material,
      defense: 100,
    };
  }
}
```
Giải thích:
- UserBuilder: khi user tạo ra item thì chỉ số sẽ là ngẫu nhiên
- TesterBuilder: khi admin/tester tạo ra item thì chỉ số sẽ là tối đa

Mình sẽ dùng Director cho ví dụ này luôn để tránh dài dòng:
```typescript
class EquipmentDirector {
  private builder: EquipmentCraft;

  constructor(builder: EquipmentCraft) {
    this.builder = builder;
  }

  buildEquipment(type: string) {
    switch (type) {
      case 'Gladiator':
        this.builder.craftMainWeapon('Spear');
        this.builder.craftArmor('Chain Armor');
        this.builder.craftGreaves('Leather Greaves');
        this.builder.craftSubWeapon('Shield');
        break;
      case 'Summoner':
        this.builder.craftMainWeapon('Magic Wand');
        this.builder.craftArmor('Robe');
        this.builder.craftGreaves('Cloth Greaves');
        break;
      default:
        break;
    }
    return this.builder.getEquipment();
  }

  changeBuilder(builder: EquipmentCraft) {
    this.builder = builder;
  }
  
  resetBuilder() {
    this.builder.reset();
  }
}
```
Do chúng ta build bộ trang bị theo class nhân vật nên sẽ tái sử dụng lại 1 phần của ví dụ trước. Sau khi khởi tạo xong giờ chúng ta chỉ cần thực hiện là xong:
```typescript
const userBuilder = new UserBuilder();
const equipmentDirector = new EquipmentDirector(userBuilder);
const userCreateGladiator = equipmentDirector.buildEquipment('Gladiator');
console.log({ userCreateGladiator });
equipmentDirector.resetBuilder();
const userCreateSummoner = equipmentDirector.buildEquipment('Summoner');
console.log({ userCreateSummoner });
equipmentDirector.resetBuilder();

const testerBuilder = new TesterBuilder();
equipmentDirector.changeBuilder(testerBuilder);
const testerCreateGladiator = equipmentDirector.buildEquipment('Gladiator');
console.log({ testerCreateGladiator });
equipmentDirector.resetBuilder();
const testerCreateSummoner = equipmentDirector.buildEquipment('Summoner');
console.log({ testerCreateSummoner });
```
Cùng xem kết quả ở console:
![image.png](https://images.viblo.asia/de014a01-9275-4623-8f5c-aa5a5c4993f6.png)

Kết quả đáp ứng được yêu cầu ban đầu mà chúng ta hướng tới.

## Áp dụng vào thực tế với dự án NestJS như thế nào? 😼
Vấn đề: Có thể mọi người sẽ thắc mắc rằng Javascript/Typescript có **Object Destructuring (OD)** nên việc ứng dụng Builder Design Pattern có thật sự cần thiết không?

Câu trả lời phụ thuộc vào việc mọi người muốn chọn đơn giản hay chọn chuyên nghiệp, dễ maintain:
- **OD** thích hợp cho các object đơn giản, nếu object phức tạp sẽ nhìn dài dòng hơi rối mắt.
- **Builder** giúp code trở nên dễ quản lý hơn vì phân chia việc tạo object thành các trường hợp cụ thể. 
 
Mình sẽ đưa ra ví dụ về cả 2 cách dùng để mọi người có thể hình dung. Bắt đầu với đoạn code Javascript sau:
```typescript
// Tạo ra ngôi nhà gỗ
const bungalow = new House({
    wall: 'wood',
    yard: 'grass',
    ceiling: 'wood',
    floor: 'wood',
})

// Tạo ra ngôi nhà kiến trúc lâu đài
const castle = new House({
    wall: 'stone',
    yard: 'flower',
    ceiling: 'stone',
    floor: 'stone',
    statue: 'monkey king',
    fence: 'brick'
})

// Tạo ra ngôi nhà kiến trúc hiện đại
const modern_hourse = new House({
    wall: 'brick',
    yard: 'swimming pool',
    ceiling: 'gypsum board',
    floor: 'ceramic tiles',
    balcony: '10m2'
})
```
- Khi dùng với NestJS chúng ta sẽ:

```typescript
// Controller
create(@Req() request: RequestWithUser, @Body() create_house_dto: CreateHouseDto){
    return this.house_service.create({
        ...create_hourse_dto,
        created_by: request.user
    })
}
// Service
create(create_house_dto: CreateHouseDto) {
    // Bỏ qua validate
    if (type === HOUSE_TYPE.BUNGALOW) {
        return this.house_repository.create({
            wall: create_house_dto.wall,
            yard: create_house_dto.yard,
            ceiling: create_house_dto.ceiling,
            floor: create_house_dto.floor,
            created_by: create_house_dto.created_by
        })
    }
    if (type === HOUSE_TYPE.CASTLE) {
        return this.house_repository.create({
            wall: create_house_dto.wall,
            yard: create_house_dto.yard,
            ceiling: create_house_dto.ceiling,
            floor: create_house_dto.floor,
            statue: create_house_dto.statue,
            fence: create_house_dto.fence,
            created_by: create_house_dto.created_by
        })
    }
    if (type === HOUSE_TYPE.MODERN) {
        return this.house_repository.create({
            wall: create_house_dto.wall,
            yard: create_house_dto.yard,
            ceiling: create_house_dto.ceiling,
            floor: create_house_dto.floor,
            balcony: create_house_dto.balcony,
            created_by: create_house_dto.created_by
        })
    }
}
```
Nhìn qua thì mọi thứ khá ổn, không nhất thiết phải có Builder Desgin Pattern, giả sử nếu áp dụng thì sẽ như thế nào?
```typescript
// Controller không có gì thay đổi
// Builder Interface
interface HouseBuilderInterface {
    buildWall(material: string)
    buildYard(kind: string)
    buildCeiling(material: string)
    buildFloor(material: string)
    buildStatue(name: string)
    buildFence(material: string)
    buildBalcony(size: string)
    getResult(): House
}
// Builder Concrete
@Injectable()
class HouseBuilder implements HouseBuilderInterface {
    private house: House;
    constructor() { this.house = new House() }
    
    buildWall(material: string) { this.house.wall = material }
    buildYard(kind: string) { this.house.yard = kind }
    buildCeiling(material: string) { this.house.ceiling = material }
    buildFloor(material: string) { this.house.floor = material }
    buildStatue(name: string) { this.house.statue = name }
    buildFence(material: string) { this.house.fence = material }
    buildBalcony(size: string) { this.house.balcony = size }
    getResult() { 
        const house = this.house;
        this.house = new House();
        return house;
    }
}

// Service
constructor(@Inject('HOUSE_BUILDER') house_builder: HouseBuilderInterface) {}

create(create_house_dto: CreateHouseDto) {
    if (type === HOUSE_TYPE.BUNGALOW) {
        const house = this.house_builder
                            .buildWall(create_house_dto.wall)
                            .buildYard(create_house_dto.yard)
                            .buildCeiling(create_house_dto.ceiling)
                            .buildFloor(create_house_dto.floor)
                            .getResult()
        return this.house_repository.create(house)
    }
    if (type === HOUSE_TYPE.CASTLE) {
         const house = this.house_builder
                            .buildWall(create_house_dto.wall)
                            .buildYard(create_house_dto.yard)
                            .buildCeiling(create_house_dto.ceiling)
                            .buildFloor(create_house_dto.floor)
                            .buildStatue(create_house_dto.statue)
                            .buildFence(create_house_dto.fence)
                            .getResult()
          return this.house_repository.create(house)
    }
    if (type === HOUSE_TYPE.MODERN) {
         const house = this.house_builder
                            .buildWall(create_house_dto.wall)
                            .buildYard(create_house_dto.yard)
                            .buildCeiling(create_house_dto.ceiling)
                            .buildFloor(create_house_dto.floor)
                            .buildBalcony(create_house_dto.balcony)
                            .getResult()
          return this.house_repository.create(house)
    }
}
```
Nhìn cũng có vẻ ổn luôn 😂, code của chúng ta trở nên rõ ràng dễ đọc hơn tuy nhiên chúng ta phải đánh đổi việc nó trở nên phức tạp và dài dòng hơn 🫠. 

Chúng ta cùng thử tích hợp luôn **Director** vào xem như thế nào:
```typescript
// Controller không có gì thay đổi
// Builder Interface không có gì thay đổi
// Builder Concrete không có gì thay đổi

// Director
class HouseDirector {
    private builder: HouseBuilderInterface;
    setBuilder(builder: HouseBuilderInterface) {
        this.builder = builder
    }
    
    buildBungalow(create_house_dto) {
        this.house_builder
                .buildWall(create_house_dto.wall)
                .buildYard(create_house_dto.yard)
                .buildCeiling(create_house_dto.ceiling)
                .buildFloor(create_house_dto.floor)
    }
    
    buildCastle(create_house_dto) {
        this.house_builder
                .buildWall(create_house_dto.wall)
                .buildYard(create_house_dto.yard)
                .buildCeiling(create_house_dto.ceiling)
                .buildFloor(create_house_dto.floor)
                .buildStatue(create_house_dto.statue)
                .buildFence(create_house_dto.fence)
    }
    
    buildModernHouse(create_house_dto) {
        this.house_builder
                .buildWall(create_house_dto.wall)
                .buildYard(create_house_dto.yard)
                .buildCeiling(create_house_dto.ceiling)
                .buildFloor(create_house_dto.floor)
                .buildBalcony(create_house_dto.balcony)
    }
}

// Service
constructor(
        @Inject('HOUSE_BUILDER') house_builder: HouseBuilderInterface 
        @Inject('HOUSE_DIRECTOR') house_director: HouseDirector // Có thể tạo interface cho director
      ) {} 

create(create_house_dto: CreateHouseDto) {
    this.house_director.setBuilder(this.house_builder);
    if (type === HOUSE_TYPE.BUNGALOW) {
        this.house_director.buildBungalow(create_house_dto);
        const house = this.house_builder.getResult()
        return this.house_repository.create(house)
    }
    if (type === HOUSE_TYPE.CASTLE) {
        this.house_director.buildCastle(create_house_dto);
        const house = this.house_builder.getResult()
        return this.house_repository.create(house)
    }
    if (type === HOUSE_TYPE.MODERN) {
        this.house_director.buildModernHouse(create_house_dto);
        const house = this.house_builder.getResult()
        return this.house_repository.create(house)
    }
}
```
Khi kết hợp với **Director** thì mọi thứ càng trở nên rõ ràng hơn, có thể thấy khi nhìn vào code chúng ta rất clean, tuy nhiên số lượng code tăng lên là việc không thể tránh khỏi. Nhưng không thể vì thể mà bỏ qua việc xem xét đến quá trình cập nhật cũng như maintain. Rõ ràng có thể thấy được việc cập nhật lại logic chúng ta sẽ chỉ tập trung vào thay đổi ở từng **Concrete Builder** mà không lo bị ảnh hưởng đến các phần còn lại.

Chúng ta sẽ cùng đứng ở góc độ **S.O.L.I.D** xem code ở trên đạt được bao nhiêu tiêu chí nhé:
- **Single Responsibility Principle**: ✅️ mỗi HouseBuilder xử lý việc logic bên trong, HouseDirector xử lý logic build lên object, HouseService thì nhận data từ Controller và gọi DB.  
- **Open/Close Principle**: ✅️ Khi cần build cho dạng nhà khác ví dụ như nhà sàn thì chỉ cần thêm vào method buildStiltHouse ở Director. Hoặc khi thay đổi nguyên liệu xây nhà thì chúng ta chỉ cần update ở Builder
- **Liskov Substitution Principle**: ⚪️ không dùng đến trong trường hợp này
- **Interface Segregation Principle**: ⚪️ không dùng đến trong trường hợp này 
- **Dependency Inversion Principle**: ✅️ chúng ta có inject HouseDirector nên cũng coi như đáp ứng 1 tí 😁

# Kết luận 📝
Vậy là chúng ta đã lần lượt đi qua khái niệm và các ví dụ về Builder Design Pattern, từ đó biết được lợi ích mà nó mang lại trong dự án. Hy vọng có thể giúp các bạn trong quá trình lập trình cũng như giải quyết được các vấn đề đã và đang gặp phải.

Nếu có thắc mắc hoặc vấn đề gì mọi người có thể comment trực tiếp ở bài viết nhé. Cảm ơn mọi người đã dành thời gian đọc bài viết 😇
# Tài liệu tham khảo 🔍
- Tran, V., go-design-pattern: Builder Pattern Solution. [online] GitHub. Available at: https://github.com/viettranx/go-design-pattern/tree/main/creational/builder/solution/builder [Accessed 25 May 2025].
- Ông Dev, 2022. Builder Pattern trong lập trình Golang (Go Design Pattern Series #1). [video] YouTube. Available at: https://www.youtube.com/watch?v=hcA7cT0_BeE [Accessed 25 May 2025].
- DigitalOcean, 2022. Builder Design Pattern in Java. [online] DigitalOcean. Available at: https://www.digitalocean.com/community/tutorials/builder-design-pattern-in-java [Accessed 25 May 2025].
- Refactoring Guru, Builder. [online] Available at: https://refactoring.guru/design-patterns/builder [Accessed 25 May 2025].
# Change log 📓
- September 07, 2024: Init document.
- May 26, 2025: Publish document