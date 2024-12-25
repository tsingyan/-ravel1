用laravel11 容器  去写示例，需体现：工厂、接口类、抽象类、门面。
---

## **目录结构**

1. **Contracts/InterviewInterface.php** (接口类)
2. **Abstracts/BaseInterview.php** (抽象类)
3. **Services/OnsiteInterview.php** 和 **Services/RemoteInterview.php** (实现类)
4. **Factories/InterviewFactoryInterface.php** (工厂接口)
5. **Factories/OnsiteFactory.php** 和 **Factories/RemoteFactory.php** (工厂实现)
6. **Factories/InterviewFactoryManager.php** (抽象工厂管理器)
7. **Facades/Interview.php** (门面类)
8. **Providers/InterviewServiceProvider.php** (服务提供者)
9. **Http/Controllers/InterviewController.php** (控制器)
10. **routes/web.php** (路由)

---

### **1. Contracts/InterviewInterface.php**
定义面试接口类：

```php
namespace App\Contracts;

interface InterviewInterface
{
    public function methodInterview(): string;
}
```

---

### **2. Abstracts/BaseInterview.php**
抽象类实现部分通用逻辑：

```php
namespace App\Abstracts;

use App\Contracts\InterviewInterface;

abstract class BaseInterview implements InterviewInterface
{

    abstract public function methodInterview(): string;
}
```

---

### **3. Services/LocalInterview.php 和 RemoteInterview.php**
具体面试子类分别实现 `methodInterview` 方法：

#### **LocalInterview**
```php
namespace App\Services;

use App\Abstracts\BaseInterview;

class LocalInterview extends BaseInterview
{
    public function methodInterview(): string
    {
        return "到场面：JJJJJJJ + YYYYYY";
    }
}
```

#### **RemoteInterview**
```php
namespace App\Services;

use App\Abstracts\BaseInterview;

class RemoteInterview extends BaseInterview
{
    public function methodInterview(): string
    {
        return "远程视频面or电话面: 您好，，，，，巴拉巴拉巴拉。。。。。";
    }
}
```

---

### **4. Factories/InterviewFactoryInterface.php**
定义工厂接口：

```php
namespace App\Factories;

use App\Contracts\InterviewInterface;

interface InterviewFactoryInterface
{
    public function createInterview(): InterviewInterface;
}
```

---

### **5. Factories/OnsiteFactory.php 和 RemoteFactory.php**
具体工厂实现：

#### **OnsiteFactory**
```php
namespace App\Factories;

use App\Contracts\InterviewInterface;
use App\Services\LocalInterview;

class LocalInterviewFactory implements InterviewFactoryInterface
{
    public function createInterview(): InterviewInterface
    {
        return new LocalInterview();
    }
}
```

#### **RemoteFactory**
```php
namespace App\Factories;

use App\Contracts\InterviewInterface;
use App\Services\RemoteInterview;

class RemoteFactory implements InterviewFactoryInterface
{
    public function createInterview(): InterviewInterface
    {
        return new RemoteInterview();
    }
}
```

---

### **6. Factories/InterviewFactoryManager.php**
管理

```php
namespace App\Factories;

class InterviewFactoryManager
{
    public function getFactory(string $type): InterviewFactoryInterface
    {
        return match ($type) {
            'local' => new LocalInterviewFactory(),
            'remote' => new RemoteFactory(),
            default => throw new \InvalidArgumentException("Invalid factory type: {$type}"),
        };
    }
}
```

---

### **7. Facades/Interview.php**
门面类封装工厂管理器的访问：

```php
namespace App\Facades;

use Illuminate\Support\Facades\Facade;

class Interview extends Facade
{
    protected static function getFacadeAccessor()
    {
        return 'interview.factory.manager';
    }
}
```

---

### **8. Providers/InterviewServiceProvider.php**
注册服务到容器以支持门面：

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Factories\InterviewFactoryManager;

class InterviewServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton('interview.factory.manager', function () {
            return new InterviewFactoryManager();
        });
    }
}
```

在 `config/app.php` 中注册服务提供者和门面别名：

```php
'providers' => [
    App\Providers\InterviewServiceProvider::class,
],

'aliases' => [
    'Interview' => App\Facades\Interview::class,
],
```

---

### **9. Http/Controllers/InterviewController.php**
控制器中调用门面获取面试对象：

```php
namespace App\Http\Controllers;

use Interview;
use Illuminate\Http\Request;

class InterviewController extends Controller
{
    public function handleInterview(Request $request)
    {
        $local = Interview::getFactory('local')->createInterview();
        $remote = Interview::getFactory('remote')->createInterview();

        return response()->json([
            'local' => $onsite->methodInterview(),
            'remote' => $remote->methodInterview(),
        ]);
    }
}
```

---

### **10. routes/web.php**
添加路由以测试：

```php
use App\Http\Controllers\InterviewController;

Route::get('/interview', [InterviewController::class, 'handleInterview']);
```

---

## **运行和测试**

访问 `/interview`，将返回以下 JSON 数据：

```json
{
    "onsite": "Conducting an onsite interview for John Doe.",
    "remote": "Conducting a remote interview for Jane Smith."
}
```

---

### **单元测试**

#### **测试门面调用**
```php
namespace Tests\Feature;

use Tests\TestCase;
use Interview;

class InterviewTest extends TestCase
{
    public function test_interview_facade()
    {
        Interview::shouldReceive('getFactory')
            ->with('onsite')
            ->andReturn(new class {
                public function createInterview($name)
                {
                    return new class($name) {
                        public function __construct(private $candidateName) {}
                        public function conductInterview()
                        {
                            return "Mock onsite interview for {$this->candidateName}.";
                        }
                    };
                }
            });

        $response = $this->get('/interview');

        $response->assertStatus(200)
                 ->assertJson([
                     'onsite' => 'Mock onsite interview for John Doe.',
                 ]);
    }
}
```

---

## **总结**

1. **抽象工厂模式**：用来管理和创建不同类型的面试对象。
2. **接口类**：定义标准方法以确保一致性。
3. **抽象类**：封装公共逻辑，避免重复代码。
4. **门面模式**：提供统一的访问接口，隐藏复杂的工厂实现。