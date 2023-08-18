1. 编写代码
   
>用户用手机号登录的时候，通常需要发送验证码，怎么防止用户恶意地重复发送验证码？写下你的思路和关键代码。重点是要避免误伤。
   
这个问题解决方案除了前端做按钮置灰控制外，后端还要做验证码校验和IP黑名单的控制。假设冷却时长为1分钟。
1. 验证码校验。key需带有IP标识，如：`code:xx.xx.xx`
      * 第一次发送验证码时，存入缓存。key带上IP标识，过期时间为1分钟。
      * 重复发送验证码时，根据IP去拿缓存的值，若存在，则提示重复发送
2. IP黑名单控制,如3分钟内重复请求5次，则将此IP拉黑。key需带有IP标识，如 `count:xx.xx.xx`
      * 判断count已然大于10，则不处理。否则下一步。
      * 检测到重复发送时，count加1，存入缓存中。


    # 是否在黑名单中
    boolean inBlackList = getIfInBlackList(ip);
    if (inBlackList){
        return;
    }
    # 是否重复发送,并判断是否拉黑
    boolean hasResend = checkCodeExists(ip);
    if (hasResend){
        int sendCount = incrSendCount();
        pushBlackListIfNecessary(sendCount);
    }
    # 发送验证码
    doSend();



2. 编写代码 
   
>现在有一个导出EXCEL表格数据的接口，因为数据量过大，还有关联数据查询，导出时间过长，如何解决这个问题，请写出思路和关键代码。

这个要考虑到是否是业务功能的缺陷，应尽量让查询条件更精确，比如精确到时间。如果数据量依旧很大，考虑使用异步导出来解决。
1. 给MQ推送一条导出的消息，带上查询条件。保存一条导出记录，为ready状态。
2. MQ消费者可以使用线程池的future模式来组装EXCEL数据
3. 上传到文件服务，若有必要，可将大文件压缩
4. 修改导出状态，记录导出结果和文件地址
2. 前端界面提供导出列表让导出记录回显。提供手动刷新按钮让用户手动刷新，或者做成自动刷新。

* MQ生产者
    

    if (pushExportTask(params)){
        recordExportLog(exportCode,params,REDAY);
        return true;
    }

    # 报错，导出任务推送失败

* MQ消费者

    
    List<Furture> flist = new ArrayList();
    # 计算记录条数
    int countData = queryListCountFromDb(parms);
    # 任务拆解,假定一个线程处理1w条数据
    int pageSize=100_00;
    int batch = countData/pageSize + countData%pageSize;
    # 使用ThreadPoolExecutor最好提前定义好
    for(int i = 0;i<batch;i++){
      Furture furture = executor.submit(()->{
            List<DataDTO> dataDto = queryDataFromDb(params,i,pageSize);
            return covertToExcelDataList(dataDto);
        });
      flist.add(furture);
    }
    
    // easyExcel无需定义对象的方式，构建excel数据
    List<List<Object>> dList = new ArrayList();
    for(Furture furture:flist){
        dList.add(furture.get());
    }

    // 上传文件
    Result uploadResult = uploadFileSerivce.uploadFile(dList);
    if(!uploadResult.isOk()){
        # 文件上传失败
        recordExportLog(exportCode,FAIL);
        return false;
    }
    
    recordExportLog(exportCode,SUCCESS,uploadResult.getFileUrl());
    return true;

   
3. 设计架构 
   
>对于一个每天有100万笔订单的商户平台，大概几万个商户，设计一个架构或独立的服务或集群， 每天凌晨生成前一天的每个运营商的对账单，包括总订单数、总金额、订单明细等；要求半小时左右生成完。

采用XXL-JOB+MQ的方式去实现。
1. XXL-JOB侧需筛选出对应的商户和前一天产生的订单号。记录每条订单的推送记录为READY状态，推送到MQ
2. MQ只会接收到用户和订单号，生成对账单明细后，设置推送记录状态为已完成，子状态为成功或失败。标记为已完成后若为该运营商的最后一条订单，则进行汇总，生成运营商完整的对账单。
   
   
4. 设计架构 
   
对于一个每天有100万笔订单的商户平台，需要增加多种支付方式（微信、支付宝、各种独立第 三方、钱包），支付到独立的商户 （每个商户都可以配置自己的收款账户），需要架构灵活，以
   及对账方便，可热更新切换，写出关键的设计思路，可以提供一些伪代码。

可以采用策略和模板方法模式实现多种支付方式，接入Apollo配置中心使用以支持热更新切换。
1. 需实现支付和对账接口
2. 统一设计热更新方法为模板方法。

* 使用方式


      # 支付
      getProcessor(PayTypeEnum.WX).pay(params);
      # 对账
      getProcessor(PayTypeEnum.WX).checkAccount(params);

* 实现，以WX为例

      
      public class WxProcessorService extends AbstractProcessorService{
        
         public Result pay(PayParams params){
            # 处理支付逻辑
         }

         public Result checkAccount(PayParams params){
            # 处理对账逻辑
         }
         
         # 支付方式
         public PayTypeEnum getPayType(){
            return PayTypeEnum.WX;
         }
      }
   
*  抽象类 `AbstractProcessorService`
   
         
         public abstract class AbstractProcessorService<T>{

            @Value("${app.default.payType}")
            private String defaultPayType;
            
            # 支付
            protected abstract Result pay(T params);
   
            # 对账
            protected abstract Result checkAccount(T params);
   
            # 支付方式
            protected abstract PayTypeEnum getPayType();
            
            public AbstractProcessorService getProcessor(PayTypeEnum payType){
               PayTypeEnum defaultType = PayTypeEnum.codeOf(defaultPayType);
               if(defaultType != null){
                  return (AbstractProcessorService)SpringUtil.getBean(defaultType.getTargetClass);
               }
               # 错误处理
            }

      }

5. 设计架构 
   
>设计可以支撑1000万设备连接的物联网平台架构，描述技术选型，以及背后的逻辑。没有经历过 可以假设，这种场景也类似直播间/游戏/聊天，海量连接的情况下如何让系统更健壮，具有扩展性、可观测性。

设备信息可以用clickHouse存储，Redis用于存储在线的设备号，MQ用于非核心操作，如日志搜集
