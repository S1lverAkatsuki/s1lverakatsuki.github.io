# 初探 MHWI LuaF 框架 Mod 开发

基于本征的 [LuaFramework](https://github.com/eigeen/LuaFramework) 编写。*原作者就在触手可及的地方，为什么不用呢*
目前写出的结果在 [MHWI-LuaF-mods](https://github.com/S1lverAkatsuki/MHWI-LuaF-mods/) 这个仓库，Mod 适用于 MHW **15.23** 版本，截至成稿期间应该没有别的更新了。*除非往里塞物语 3 的广告*

## 开发思路

我想实现的功能是：**当怪物血量小于某个定值的时候，打印一个提醒我该超解收尾的系统提示**。

![提示](/posts/imgs/s1aT31FA4LJuSuu9P1yEbu.png)

拿到库以后，就决定去找有没有相关的 API，LuaF 并没有提供直接的「访问怪物血量」的函数。
试着去查一下 `sdk.d.lua`。

- `LuaPtr` 是游戏内的指针对象的脚本对照。下面很多按照某个数据类型读取它的方法。
  一个很常见的应用是判断指针是不是合法的：

  ```lua
  local function is_valid_ptr(ptr)
    return ptr ~= nil and ptr:to_integer() ~= 0
  end
  ```

  ---

  既然是指针，肯定还有增减偏移的方法。

  ```lua
  ptr:offset(42)  -- 返回指向 ptr + 0x42 的 LuaPtr 对象
  ```

- `Monster` 在我目前的开发中主要用了 `list` 字段，获取当前地图里的怪物列表，第二个返回值为怪物基址

  ```lua
  local monsters = sdk.Monster.list()
  for _, addr in pairs(monsters) do
      local ptr = sdk.LuaPtr(addr)
      -- ptr 是可在脚本内操作的怪物对象基址
  end
  ```

此外还有一些别的方法：

- 判断玩家是否在地图内：

  ```lua
  local Common = require("_framework.game.common")
  Common:is_player_in_scene() -- 在图内返回真
  ```

- 打印系统消息（效果参照截图，只有装 Mod 的人才能看到）

  ```lua
  local Message = require("_framework.game.message")
  Message.show_system("要打印的信息", Message.SystemMessageColor.Purple)
  -- 只有蓝色（Message.SystemMessageColor.Blue）和紫色
  ```

## 寻找内存

有了 API 以后，然后到了逆向阶段。
使用 CE 查找血量，但是无法确认它相对于怪物基址（可用 LuaF 提供的方法获取）的偏移。*大模型猜的全错🤖👎*
之后本征给我了个 CT 表，可以从里面获取一个地址的注释和相关的指针链。比如我正在找的怪物当前血量：

![指针链](/posts/imgs/7543W2fwiptqtQQ9QMzpRs.png)

其中的 `1360e0080` 是可以被获取的**怪物基址**
`0x7670` 和 `0x64` 是两个硬编码的偏移。
附加一些「合理的」猜测，可以认为

- `0x7670` 是怪物基址到血量模块的偏移
- `0x64` 是血量结构体到当前血量的偏移

如果用 $\left[0x42\right]$ 来表示访问地址 `0x42` 的值，我们可以写成：

$$
\text{当前血量} = \left[\left[\text{怪物基址} + 0x7060\right] + 0x64\right]
$$

之后就可以以此写出脚本了，注意当前血量的数据类型是**单精度浮点型**（f32），用对读取方法。

---

使用偏移定位，那要是版本更新以后呢，多一条汇编偏移就没法使用了。
所以就有了特征码，也就是有着关键信息的一条汇编语句，直接根据汇编语句获取。
比如 `0x7670` 可以从一条汇编语句中找到 `mov rax,[rdi+7670]`，转成十六进制可得：

```text
48 8B 87 70 76 00 00 
```

这就是特征码。
特征码可以替换里面的任意组十六进制字符为 `??` 通配符。
比如 `48 8B 87 ?? ?? 00 00`，这就是匹配 `mov rax,[rdi+<任意四个十六进制字符>]`
可以参照 `AddressRepository` 进行调用：

```lua
local hp_module_offset = sdk.AddressRepository.get_or_insert(
    "HPModule",
    "48 8B 87 70 76 00 00",
    3 -- 偏移量，代表从第 4 个字节开始开始
):read_u32()
```

本征的建议：
> 如果操作内容包含字段偏移之类的也可以不通配，反正偏移炸了，代码其他地方也会炸。
> 基本上 call 是一定要通配的，其他的看情况。

## Lua 补充

*毕竟是第一次写*

- `local` 表明是局部变量，不加全局，无脑加就好了
- 访问运算符用 `:` 而不是正常编程语言的 `.`
- 反人类 1-based 索引
- `number` 类型只是双精度浮点
- 只有字典 `table` 和数组（没有值的表）这俩高级数据类型，都用大括号声明
- 代码层级采用 `do`（`for` 和 `while` 循环条件后面）、`then`（`if` 和 `elseif` 条件后面）和 `end` 分割，一股 Fortran 味道
- 有 `break`，但是没有 `continue`，需要靠 goto 实现：

  ```lua
  for i in 1..10 do
    if i == 5 then
      -- 索引等于 5 的时候跳过剩下的部分
      goto continue
    end
    -- 其他部分
    ::continue::
  end
  ```

- `~=` 就是别的语言的 `!=`
- `#` 是取长度符号，可以取字符串或是表的长度（表中间不连续索引值未知）

## 一些可以用的数据

### 偏移

| 源 | 偏移 | 目的 |
| -- | :----: | ----: |
| 怪物对象基址 | + `0x12280` | 当前怪物的 ID |
| 怪物对象基址 | + `0x7670` | 当前怪物的血量模块 |
| 血量模块 | + `0x60` | 当前怪物最大血量 |
| 血量模块 | + `0x64` | 当前怪物血量 |

### 怪物 ID

直接用 Lua 代码方便复制

```lua
local largeMonsterRanges = {
    -- 本体怪物
    [0x00] = true, -- 蛮颚龙
    [0x01] = true, -- 火龙
    [0x04] = true, -- 熔山龙
    [0x07] = true, -- 大贼龙
    [0x09] = true, -- 雌火龙
    [0x0A] = true, -- 樱火龙
    [0x0B] = true, -- 苍火龙
    [0x0C] = true, -- 角龙
    [0x0D] = true, -- 黑角龙
    [0x0E] = true, -- 麒麟
    [0x0F] = true, -- 贝希摩斯（联动）
    [0x10] = true, -- 钢龙
    [0x11] = true, -- 炎妃龙
    [0x12] = true, -- 炎王龙
    [0x13] = true, -- 熔岩龙
    [0x14] = true, -- 恐暴龙
    [0x15] = true, -- 土砂龙
    [0x16] = true, -- 爆锤龙
    [0x17] = true, -- 鹿首精（联动）
    [0x33] = true, -- 古代鹿首精（联动）
    [0x18] = true, -- 毒妖鸟
    [0x19] = true, -- 灭尽龙
    [0x1A] = true, -- 冥灯龙
    [0x1B] = true, -- 骚鸟
    [0x1C] = true, -- 眩鸟
    [0x1D] = true, -- 泥鱼龙
    [0x1E] = true, -- 飞雷龙
    [0x1F] = true, -- 浮空龙
    [0x20] = true, -- 风漂龙
    [0x21] = true, -- 大痹贼龙
    [0x22] = true, -- 惨爪龙
    [0x23] = true, -- 骨锤龙
    [0x24] = true, -- 尸套龙
    [0x25] = true, -- 岩贼龙
    [0x26] = true, -- 绚辉龙
    [0x27] = true, -- 爆鳞龙
    
    -- 冰原怪物
    [0x3D] = true, -- 轰龙
    [0x3E] = true, -- 迅龙
    [0x3F] = true, -- 冰牙龙
    [0x40] = true, -- 惶怒恐暴龙
    [0x41] = true, -- 碎龙
    [0x42] = true, -- 斩龙
    [0x43] = true, -- 硫斩龙
    [0x44] = true, -- 雷颚龙
    [0x45] = true, -- 水妖鸟
    [0x46] = true, -- 歼世灭尽龙
    [0x47] = true, -- 痹毒龙
    [0x48] = true, -- 浮眠龙
    [0x49] = true, -- 霜翼风漂龙
    [0x4A] = true, -- 凶爪龙
    [0x4B] = true, -- 雾瘴尸套龙
    [0x4C] = true, -- 红莲爆鳞龙
    [0x4D] = true, -- 冰鱼龙
    [0x4E] = true, -- 猛牛龙
    [0x4F] = true, -- 冰呪龙
    [0x50] = true, -- 溟波龙
    [0x51] = true, -- 天地煌啼龙
    
    -- 免费更新怪物
    [0x57] = true, -- 煌黑龙
    [0x58] = true, -- 金火龙
    [0x59] = true, -- 银火龙
    [0x5A] = true, -- 黑狼鸟
    [0x5B] = true, -- 金狮子
    [0x5C] = true, -- 激昂金狮子
    [0x5D] = true, -- 黑轰龙
    [0x5E] = true, -- 雷狼龙
    [0x5F] = true, -- 狱狼龙
    [0x60] = true, -- 猛爆碎龙
    [0x61] = true, -- 冥赤龙
    [0x63] = true, -- 战痕黑狼鸟
    [0x64] = true, -- 霜刃冰牙龙
    [0x65] = true, -- 黑龙
}
```
