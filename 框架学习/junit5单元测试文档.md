

测试的编写大概分成4个部分

1. 测试数据文件
2. 整合测试数据的枚举
3. 测试数据的转换
4. 测试方法

### 1.测试数据文件

支持json文件和excel文件，统一放到**test/resources/file/**下，可按不同功能建立不同的文件夹区分。

json的数据格式即为对应测试方法参数的数据格式

目录样例：

```
#根据不同的功能建立不同的路径，每个文件夹放针对该功能的各种情况的测试用例
file
 ├─spu
 │  ├─add
 │  │      spuAddTestCustomInfo.json
 │  │      spuAddTestNewSkuAdvanceConfigCustomInfo.json
 │  │      spuAddTestNewSkuAdvanceConfigOtherInfo.json
 │  │      spuAddTestNewSkuAdvanceConfigSizeInfo.json
 │  │      spuAddTestNewSkuDeclareInfo.json
 │  │      spuAddTestNewSkuNameException.json
 │  │      spuAddTestNewSkuNoException.json
 │  │      spuAddTestSpuNameException.json
 │  │      spuAddTestSpuNoException.json
 │  │
 │  ├─delete
 │  │      testDelete.json
 │  │
 │  ├─import
 │  │      基础测试.xls
 │  │
 │  └─update
 │          spuUpdateTestCustomInfo.json
 │          spuUpdateTestNewSkuAdvanceConfigCustomInfo.json
 │          spuUpdateTestNewSkuAdvanceConfigOtherInfo.json
 │          spuUpdateTestNewSkuAdvanceConfigSizeInfo.json
 │          spuUpdateTestNewSkuDeclareInfo.json
 │          spuUpdateTestNewSkuNameException.json
 │          spuUpdateTestSpuNameException.json
 │          spuUpdateTestSpuNoException.json

```

json示例

```json
{
  "brandId": 1,
  "classId": 14,
  "currency": "CNY",
  "goodsNo": "",
  "sellerId": -1,
  "goodsName": "ceshi0819skuName-9",
  "remark": "0817",
  "flagId": 10018,
  "specList": [
    {
      "specNo": "ceshi0819skuNo-9",
      "specName": "ceshi0819skuName-9",
      "imgUrl": "",
      "barcode": "",
      "retailPrice": "",
      "currency": "CNY",
      "declareNameCn": "",
      "declareNameEn": "",
      "hsCode": "",
      "declarePrice": "",
      "declareWeight": "",
      "length": "",
      "width": "",
      "height": "",
      "weight": "",
      "largeType": 0,
      "isNotNeedExamine": 0,
      "remark": "",
      "aliProperties": 0,
      "goodsProperty": 0,
      "prop1": "",
      "prop2": "",
      "prop3": "",
      "prop4": "",
      "prop5": "",
      "prop6": "",
      "id": 0,
      "tagList": [

      ],
      "visible1": false,
      "visible": false
    }
  ],
  "prop1": "",
  "prop2": "",
  "prop3": "",
  "prop4": "",
  "prop5": "",
  "prop6": "",
  "addPageType": 1
}
```

### 2.整合测试数据的枚举

每个枚举类对应要测试的功能，枚举的多个实例对应要测试的不同情况，**必须要继承*ITestDataEnum*接口**（json数据转换用）

*testFilePath*表示测试文件相对路径，每个枚举实例上最好加上对应测试情况的描述，以便可直观看出该功能的测试点

```java
/**
 * 组合装测试新增数据
 */
public enum SuiteAddTestEnum implements ITestDataEnum {
    /**
     * 组合装测试
     */
    //组合装明细申报无自定义信息
    SUITE_ADD_TEST_DETAIL_DECLARE_NO_DEFINE("file/suites/add/suiteAddTestDetailDeclareNoDefine.json"),
    //组合装明细申报有自定义信息
    SUITE_ADD_TEST_DETAIL_DECLARE_YES_DEFINE("file/suites/add/suiteAddTestDetailDeclareYesDefine.json"),
    //组合装申报无自定义信息
    SUITE_ADD_TEST_DECLARE_NO_DEFINE("file/suites/add/suiteAddTestDeclareNoDefine.json"),
    //组合装申报有自定义信息
    SUITE_ADD_TEST_DECLARE_YES_DEFINE("file/suites/add/suiteAddTestDeclareYesDefine.json"),
    //组合SKU编码异常
    SUITE_ADD_TEST_SKU_NO_EXCEPTION("file/suites/add/suiteAddTestSkuNoException.json"),
    //组合SKU名称异常
    SUITE_ADD_TEST_SKU_NAME_EXCEPTION("file/suites/add/suiteAddTestSkuNameException.json");

    private String testFilePath;

    SuiteAddTestEnum(String testFilePath) {
        this.testFilePath = testFilePath;
    }

    @Override
    public String getTestFilePath() {
        return testFilePath;
    }
}
```

### 3.测试数据的转换

方法位于cn.wangdian.convert下

枚举中存放的都是测试文件的路径，需要将测试文件转成相应对象，以便与测试方法的调用，转换的方法就是对应的XXXConverter类

目前支持

- json   => Object -----------------ToObjectArgumentConverter.java
- json   => Collection -------------ToCollectionArgumentConverter.java
- excel => MultipartFile ----------ToMultipartFileArgumentConverter.java

convert用法示例

用法：

```java
//参数化注解
@ParameterizedTest
//测试数据存放的枚举类，若不指定names则该枚举的所有实例都会执行
@EnumSource(value = SuiteUpdateTestEnum.class, names = {"SUITE_UPDATE_TEST_ITEM_INFO"}) 
//ToObjectArgumentConverter可将对应枚举实例中的文件路径转成目标对象
public void test(@ConvertWith(ToObjectArgumentConverter.class) GoodsSuiteParam param)
```

多个测试参数的方法可参考官网的实例（如下），简单的单参数类型可使用**@ValueSource**

```java
@ParameterizedTest
@MethodSource("stringIntAndListProvider")
void testWithMultiArgMethodSource(String str, int num, List<String> list) {
    assertEquals(5, str.length());
    assertTrue(num >=1 && num <=2);
    assertEquals(2, list.size());
}

static Stream<Arguments> stringIntAndListProvider() {
    //具体参数可放到测试文件中，然后读取文件
    return Stream.of(
        arguments("apple", 1, Arrays.asList("a", "b")),
        arguments("lemon", 2, Arrays.asList("x", "y"))
    );
}

//简单多多参数可使用@CsvSource
@ParameterizedTest
@CsvSource({"sid1,account1,18888888888","sid2,account2,18888888888"})
void testAccount(String sid, String account, String mobileNo){}
```

### 4.测试方法

junit5官方文档：https://junit.org/junit5/docs/current/user-guide/，可重点看参数化测试那章

测试类需要继承CommonTest类

#### ①常用注解

```java
@DisplayName("XXX测试") #自定义测试名称，可标注在类及方法上，主要用来简要描述测试内容
@Nested #用于嵌套测试，可作用与内部类上
@ParameterizedTest #参数化测试，支持配置参数
@EnumSource  #提供枚举作为参数，通过names可指定要测试的枚举实例，不指定默认执行全部
@ConvertWith #参数转换注解，可将@XXXSource注解提供的参数转换成目标类型
。。。
```

#### ②事务回滚方法

可通过CommonTest.rollback方法回滚数据

```java
rollback(() -> {
                //要测试的方法
                //针对测试方法的断言
});
```

#### ③断言

理论上每个测试方法都应该有个断言,一个业务方法大体上主要是操作数据库，简单的断言可直接使用junit的断言方法，复杂的断言需要对数据库进行断言，大概步骤如下：

1. 根据测试方法参数查表（一般为单表查询）
2. 每个Executable校验一个表，内部可以使用assertAll方法对具体字段断言
3. 最后使用assertAll校验上述的Executable

如果是顺序执行的断言，前面的断言失败了后续断言就不会在执行，使用assertAll的好处是即便某个断言失败了，其他的断言仍然最后会判断，这样一次执行就可以看出所有的断言结果

ps:使用Jooq查表我不确定是否合理，应该有更合适的方法

```java
@DisplayName("新增明细申报无自定义信息")
        @ParameterizedTest
        @EnumSource(value = SuiteAddTestEnum.class, names = {"SUITE_ADD_TEST_DETAIL_DECLARE_NO_DEFINE"})
        public void testDetailDeclareNoDefine(@ConvertWith(ToObjectArgumentConverter.class) GoodsSuiteParam param) {
            rollback(() -> {
                //要测试的方法
                goodsSuiteController.addGoodsSuite(param);
                //针对测试方法的断言
                testDetailDeclareNoDefineAssert(param);
            });
        }

        private void testDetailDeclareNoDefineAssert(GoodsSuiteParam param) {
            //组合装信息期望值（来自参数）
            final GoodsSuiteDTO suiteInfoFromExpected = param.getSuiteInfo();
            final List<GoodsSuiteDetailDTO> suiteDetailsFromExpected = suiteInfoFromExpected.getGoodsSuiteDetailDTOList();
            //组合装信息实际值（来自查表）
            final GoodsSuiteDTO suiteDTOFromActual = JOOQ.db().selectFrom(GOODS_SUITE)
                    .where(GOODS_SUITE.SUITE_NO.eq(suiteInfoFromExpected.getSuiteNo()))
                    .fetchOneInto(GoodsSuiteDTO.class);
            //组合装明细信息实际值（来自查表）
            final List<GoodsSuiteDetailDTO> suiteDetailDTOSFromActual = JOOQ.db().selectFrom(GOODS_SUITE_DETAIL)
                    .where(GOODS_SUITE_DETAIL.SUITE_ID.eq(suiteDTOFromActual.getSuiteId()))
                    .fetchInto(GoodsSuiteDetailDTO.class);
            //校验组合装信息的断言
            Executable suiteInfoAssert = () -> {
                assertNotNull(suiteDTOFromActual, "插入数据不应为空");
                assertAll(() -> assertEquals(suiteInfoFromExpected.getSuiteName(), suiteDTOFromActual.getSuiteName(), "组合装名称不一致"),
                        () -> assertEquals(suiteInfoFromExpected.getBarcode(), suiteDTOFromActual.getBarcode(), "组合装条码不一致")
                        //...
                );
            };
            //校验组合装明细信息的断言
            Executable suiteDetailInfoAssert = () -> {
                assertNotEquals(0, suiteDetailDTOSFromActual.size(), "未发现组合装明细");
                final GoodsSuiteDetailDTO detailFromActual = suiteDetailDTOSFromActual.get(0);
                final GoodsSuiteDetailDTO detailFromExpected = suiteDetailsFromExpected.get(0);
                assertAll(() -> assertEquals(detailFromExpected.getSpecId(), detailFromActual.getSpecId(), "组合装明细单品Id不一致")
                        //...
                );
            };
            assertAll(suiteInfoAssert, suiteDetailInfoAssert);
        }
```

#### ④测试类结构

外部类表示功能模块，内部类表示要测试的具体功能

结构示例如下

```java
@DisplayName("组合装功能测试")
public class SuiteFunctionalTest extends CommonTest {

    @Autowired
    private GoodsSuiteService goodsSuiteService;
    @Autowired
    private GoodsSuiteController goodsSuiteController;

    @Nested
    @DisplayName("组合装新增测试")
    class SuiteAddTest {

        @DisplayName("新增申报有自定义信息")
        @ParameterizedTest
        @EnumSource(value = SuiteAddTestEnum.class, names = {"SUITE_ADD_TEST_DECLARE_YES_DEFINE"})
        public void testDeclareYesDefine(@ConvertWith(ToObjectArgumentConverter.class) GoodsSuiteParam param) {
            rollback(() -> {
                goodsSuiteController.addGoodsSuite(param);
                //todo 断言
            });
        }

        @DisplayName("新增组合SKU编码异常")
        @ParameterizedTest
        @EnumSource(value = SuiteAddTestEnum.class, names = {"SUITE_ADD_TEST_SKU_NO_EXCEPTION"})
        public void testSkuNoException(@ConvertWith(ToObjectArgumentConverter.class) GoodsSuiteParam param) {
            rollback(() -> {
                Assertions.assertThrows(ConstraintViolationException.class, () -> {
                    goodsSuiteController.addGoodsSuite(param);
                    //todo 断言
                });
            });
        }
        //...
    }

    @Nested
    @DisplayName("组合装编辑测试")
    class SuiteUpdateTest {}

    @Nested
    @DisplayName("组合装删除测试")
    class SuiteDeleteTest {

        @DisplayName("批量删除")
        @ParameterizedTest
        @EnumSource(value = SuiteDeleteTestEnum.class, names = {"SUITE_DELETE_BASE_TEST"})
        public void testDeleteSuite(@ConvertWith(ToCollectionArgumentConverter.class) List<Integer> suiteIdList) {
            rollback(() -> {
                goodsSuiteController.deleteGoodsSuite(suiteIdList);
            });
        }
    }

    @Nested
    @DisplayName("组合装导入测试")
    class SuiteImportTest {}
}
```

