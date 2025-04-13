<div align=center>
    <img src="./lol">
    <br /><br />
    <p>Fork of <a href="https://github.com/PaperMC/Folia/tree/dev/1.21.4?tab=readme-ov-file">Folia-paper</a> Với Tối ưu hệ thống đa luồng theo vùng.

## Overview

Folia nhóm các khối được tải gần đó để tạo thành một "vùng độc lập".
Xem [tài liệu PaperMC](https://docs.papermc.io/folia/reference/region-logic) để biết thông tin chi tiết chính xác về cách Folia
sẽ nhóm các khối gần đó.
Mỗi vùng độc lập có vòng lặp tích tắc riêng, được tích tắc ở
tốc độ tích tắc Minecraft thông thường (20TPS). Các vòng lặp tích tắc được thực thi
trên một nhóm luồng song song. Không còn luồng chính nữa,
vì mỗi vùng thực sự có "luồng chính" riêng thực thi
toàn bộ vòng lặp tích tắc.

Đối với một máy chủ có nhiều người chơi phân tán, Folia sẽ tạo ra nhiều
vùng phân tán và đánh dấu tất cả chúng song song trên một
threadpool có kích thước có thể cấu hình. Do đó, Folia sẽ mở rộng tốt cho các máy chủ như thế này.


## FAQ

### Thể loại máy chủ nào sẽ lợi khi dùng folia ?
Các loại máy chủ phân tán người chơi một cách tự nhiên,
như skyblock hoặc SMP, sẽ được hưởng lợi nhiều nhất từ ​​Folia. Máy chủ
cũng phải có số lượng người chơi đáng kể.
### Phần cứng tốt nhất để chạy ?
Tốt nhất sẽ là 16 _nhân_ (Không phải luồng) hoặc 32 _luồng_

### Làm sao để cấu hình Folia tốt nhất?
Đầu tiên, nên load trước thế giới để giảm đáng kể số lượng luồng công việc hệ thống khối cần thiết.

Sau đây là ước tính _rất sơ bộ_ dựa trên thử nghiệm
được thực hiện trước khi Folia được phát hành trên máy chủ thử nghiệm mà chúng tôi chạy
có ~330 người chơi đạt đỉnh. Vì vậy, nó không chính xác và sẽ cần điều chỉnh thêm -
chỉ coi đó là điểm khởi đầu. Cre : paper team

Tổng số lõi trên máy có sẵn phải được
tính đến. Sau đó, phân bổ luồng cho:
- netty IO/thread :~4 per 200-300 players
- chunk system io threads: ~3 per 200-300 players
- chunk system workers Nếu world được load trước, ~2 per 200-300 players
- Không có dự đoán tốt nhất cho các công nhân hệ thống khối nếu không được tạo trước, vì
trên máy chủ thử nghiệm mà chúng tôi chạy, chúng tôi đã đưa ra 16 luồng nhưng việc tạo khối vẫn
chậm ở mức ~300 người chơi. Cre : paper team
- Cấu hình GC: ???? Nhưng, GC cài đặt _do_ phân bổ các luồng đồng thời, và bạn cần
biết chính xác có bao nhiêu. Điều này thường thông qua cờ `-XX:ConcGCThreads=n`. Đừng
nhầm lẫn cờ này với `-XX:ParallelGCThreads=n`, vì các luồng GC song song chỉ chạy khi
ứng dụng bị GC tạm dừng và do đó không nên tính đến.

Sau khi phân bổ xong, các lõi còn lại trên hệ thống cho đến khi phân bổ 80% (tổng số luồng được phân bổ < 80% CPU khả dụng) có thể được phân bổ cho tickthreads (theo cấu hình toàn cục, threaded-regions.threads).
The reason you should not allocate more than 80% of the cores is due to the
thực tế là các plugin hoặc thậm chí máy chủ có thể sử dụng các luồng bổ sung
mà bạn không thể cấu hình hoặc thậm chí dự đoán.

Ngoài ra, tất cả những điều trên chỉ là phỏng đoán sơ bộ dựa trên số lượng người chơi, nhưng
rất có thể việc phân bổ luồng sẽ không lý tưởng và bạn
sẽ cần điều chỉnh dựa trên việc sử dụng các luồng mà cuối cùng bạn thấy.

## Plugin Hỗ trợ

Không còn luồng chính nào nữa. Tôi mong đợi _mọi_ plugin duy nhất
tồn tại đều yêu cầu _một số_ mức độ sửa đổi để hoạt động
trong Folia. Ngoài ra, đa luồng _bất kỳ loại nào_ cũng sẽ tạo ra
các điều kiện chạy đua có thể xảy ra trong dữ liệu được plugin lưu giữ - vì vậy, chắc chắn
sẽ có những thay đổi cần được thực hiện.

Vì vậy, hãy đặt kỳ vọng của bạn về khả năng tương thích ở mức 0.


### Current API additions

To properly understand API additions, please read
[Project overview](https://docs.papermc.io/folia/reference/overview).

- RegionScheduler, AsyncScheduler, GlobalRegionScheduler, and EntityScheduler 
  acting as a replacement for  the BukkitScheduler.
  The entity scheduler is retrieved via Entity#getScheduler, and the
  rest of the schedulers can be retrieved from the Bukkit/Server classes.
- Bukkit#isOwnedByCurrentRegion to test if the current ticking region
  owns positions/entities

### Thread contexts for API

To properly understand API additions, please read
[Project overview](https://docs.papermc.io/folia/reference/overview).

General rules of thumb:

1. Commands for entities/players are called on the region which owns
the entity/player. Console commands are executed on the global region.

2. Events involving a single entity (i.e player breaks/places block) are
called on the region owning entity. Events involving actions on an entity
(such as entity damage) are invoked on the region owning the target entity.

3. The async modifier for events is deprecated - all events
fired from regions or the global region are considered _synchronous_, 
even though there is no main thread anymore. 

### Current broken API

- Most API that interacts with portals / respawning players / some
  player login API is broken.
- ALL scoreboard API is considered broken (this is global state that
  I've not figured out how to properly implement yet)
- World loading/unloading
- Entity#teleport. This will NEVER UNDER ANY CIRCUMSTANCE come back, 
  use teleportAsync
- Could be more

### Planned API additions

- Proper asynchronous events. This would allow the result of an event
  to be completed later, on a different thread context. This is required
  to implement some things like spawn position select, as asynchronous
  chunk loads are required when accessing chunk data out-of-region.
- World loading/unloading
- More to come here

### Planned API changes

- Super aggressive thread checks across the board. This is absolutely
  required to prevent plugin devs from shipping code that may randomly
  break random parts of the server in entirely _undiagnosable_ manners.
- More to come here

### Maven information
* Maven Repo (for folia-api):
```xml
<repository>
    <id>papermc</id>
    <url>https://repo.papermc.io/repository/maven-public/</url>
</repository>
```
* Artifact Information:
```xml
<dependency>
    <groupId>dev.folia</groupId>
    <artifactId>folia-api</artifactId>
    <version>1.20.1-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
 ```


## License
The PATCHES-LICENSE file describes the license for api & server patches,
found in `./patches` and its subdirectories except when noted otherwise.

The fork is based off of PaperMC's fork example found [here](https://github.com/PaperMC/paperweight-examples).
As such, it contains modifications to it in this project, please see the repository for license information
of modified files.
