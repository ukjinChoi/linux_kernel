# Platform Devices and Drivers
<linux/paltform_device.h>에는 platform bus: platform_device, platform_driver 같은 driver model interface를 확인할 수 있다.
이 pseudo-bus는 system-on-chip 에서 peripheral을 통합하는 것들 또는 몇 "legacy" PC interconnects; PCI나 USB 같이 확장성 있는;처럼 작은 infrastructure와 함께 버스에 device를 연결하는데 사용된다.

## Platform devices
Platform devices는 devices인데, 시스템에서 자율적인 주체이다. 이것은 legacy port-base devices 와 peripheral buses로 host bridge, system-on-chip platform에 통합된 대부분의 컨트롤러를 포함한다. 공통적으로 가지고 있는 것은 CPU bus에서부터 direct addressing이다. 드물게, platform_device는 다른 종유의 영역을 통해 연결될 수 있다;하지만 여전히 register는 직접 접근 가능하다.

Platform devices는 name이 있고, drver binding으로 사용되며, addresses와 IRQs같은 리소스가 주어진다.
``` c
struct platform_device {
    const char * name;
    u32 id;
    struct device dev;
    u32 num_resources;
    struct resource * resource;
};
```

## Platform drivers
Platform drivers는 표준 driver model convention을 따르며, discovery/enumeration은 drivers 외부에서 다뤄지고, drivers는 probe()와 remove() methods를 제공한다. 표준 conventions을 사용하여 power management와 shutdown notifcations을 지원한다.

``` c
struct platform_driver {
    int (*probe)(struct platform_device*);
    int (*remove)(struct platform_device*);
    void (*shutdown)(struct platform_device*);
    int (*suspend)(struct platform_device*, pm_message_t state);
    int (*suspend_late)(struct paltform_device*, pm_message_t state);
    int (*resume_early)(struct platform_device*);
    int (*resume)(struct platform_device*);
    struct device_driver driver;
};
```

probe()는 명시된 device hardware가 실제 존재하는지를 검증해야한다; 가끔 platform setup code는 확신할 수 없다. probing은 clocks, device platform_data를 포함하는 device resource를 사용할 수 있다.

platform driver register 는 다음처럼 등록한다.
``` c
int platform_driver_register(struct platform_driver* drv);
```
또는 hot-pluggable이지 않으면, probe() routine은 driver의 runtime memory footprint를 감소시키기 위해 init section에 있을 수 있다.

``` c
int platform_driver_probe(struct platform_driver *drv, int(*probe)(struct platform_device*));
```

Kernel modules는 몇가지 platform driver의 복합일 수 있다. platform core는 drivers 배열을 register, unregister하도록 지원한다.

``` c
int __platform_register_drivers(struct platform_driver* const * drivers, unsigned int count, struct module* owner);
void platform_unregister_drivers(struct platform_driver* const * drivers, unsigned int count);
```

만약 drivers 중 하나가 register 실패하면, 해당 시점까지 등록된 모든 drivers 는 반대 순서로 unregistered 된다. owner paramter로 THIS_MODULE를 전달하는 편리한 macro가 있다.

``` c
#define platform_register_drivers(drivers, count)
```

## Device Enumeration

일반적으로, platform specific (and often board-specific) setup code는 platform devices를 register한다.
``` c
int platform_device_register (struct platform_device* pdev);
int platform_add_devices (struct platform_device** pdevs, int ndev);
```
일반적인 규칙은 실제로 존재하는 deivce만 등록하는 것이지만, 경우에 따라 추가 장치가 등록될 수 있다. 예로, 커널은 모든 보드에 등록되지 않을 수 있는 외부 네트워크 어댑터와 함께 작동하도록 구성되거나 일부 보드가 peripheral에 연결할 수 없는 통합 컨트롤러와 함께 작동하도록 구성될 수 있다.

경우에 따라, boot firmware는 주어진 보드에 등록된 device를 설명하는 table을 export한다. 이러한 table이 없으면, 시스템 설정 코드가 올바른 장치를 설정할 수 있는 유일한 방법은 특정 대상 보드용 커널을 구축하는 것 뿐이다. 이러한 보드 전용 커널은 임베디드 및 맞추형 시스템 개발에서 흔히 볼 수 있다.

platform device와 연관된 memory와 IRQ resouces만으로 device driver 동작이 충분하지 않은 경우가 많다. 보드 설정 코드는 가끔 장치의 platform-data field를 사용하여 추가 정보를 제공하여 추가 정보를 보관한다.

임베디드 시스템에는 플랫폼 장치에 하나 이상의 clock이 필요한 경우가 많으며, 일반적으로 (전력 절약을 위해) 필요할 때까지 꺼둔다. 또한 시스템 설정은 이러한 clock를 장치와 연결하여 필요에 따라 clk_get(&pdev->dev, clock_name)을 호출하여 clock을 반환한다.

## Legacy Drivers: Device Probing
일부 드라이버는 비드라이버 역할을 가지기 때문에 드라이버 모델로 완전히 전환되지 않는 경우가 있다. 드라이버는 시스템 인프라에 맡기는 대신 플랫폼 장치를 등록한다. 이러한 메커니즘을 사용하면 디바이스를 드라이버와 다른 시스템 구성 요소에 만들어야 하므로 이러한 드라이버를 핫플러그 하거나 콜드플러그 할 수 없다.

이 스타일은 권장되지 않는다. 

``` c 
struct platform_device* platform_device_alloc(const char* name, int id);
```

platform_device_alloc을 사용하여 디바이스를 동적으로 할당한 다음 리소스와 platform_device_register()로 초기화할 수 있다.

## Device Naming and Driver Binding

platform_device.dev.bus_id는 디바이스 표준 이름이다. 두 가지 요소로 구성된다.

* platform_device.name ... which is also used to for driver matching.
* platform_device.id ... the device instance number, or else "-1" to indicate ther's only one.

Driver binding은 driver core에 의해 자동으로 수행된다. device와 driver가 일치함을 찾은 후 driver probe()를 호출한다. 만약 probe()가 성공하면, driver와 device는 정상적으로 binding 된다. 일치함을 찾는 방법은 세 가지 방법이 있다.

* device가 등록될 때마다 bus에 해당하는 driver는 체크한다. Platform driver는 system boot시 일찍 등록해야 한다.
* platform_driver_register()를 사용하여 driver가 등록될 때, 모든 unbound device는 검사된다. Driver는 주로 boot 중 나중에 또는 module loading으로 등록된다.
* platform_driver_probe()를 사용하여 driver를 등록하는 것은 platform_driver_register()와 유사하지만, 만약 다른 device가 등록되어 있으면 나중에 드라이버를 probe하지 않는 점이 다르다.