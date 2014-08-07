
## 自夸 ##

### 前端请求： @Valid + Validator ###

CampaignController CampaignForm

@Valid Simple Valide方式 + 复杂校验方式

业务处理方专注于业务校验。

### 前端请求： @PartFormValidate 支持部分校验方案 ###

CampaignController CampaignForm ModifyFormAspect RequestFormBase

打破SpringMvc只支持全校验的方式。

### 后端返回： 通用列表返回结构 与 通用分页 ###

DaoPageResult

1. 业务处理方无需要关注请求参数。（分页信息使用page2方法自动进行）
2. 业务处理方无需要关注返回参数。（类似footer, totalcount, pageno, pagesize 均不用开发者关心）
3. 业务处理方关注数据。只需要实现转换方法。

		/**
		 * 新版的Page
		 * 
		 * @param matches
		 * @param daoPage
		 * @return
		 */
		public DaoPageResult<ENTITY> page2(List<Match> matches, DaoPage daoPage) {
		
		    DaoPageResult<ENTITY> daoPageResult = new DaoPageResult<ENTITY>();
		
		    // 结果
		    List<ENTITY> result = page(matches, daoPage);
		    daoPageResult.setResult(result);
		
		    // 总数
		    int totalCount = count(matches);
		    daoPageResult.setTotalCount(totalCount);
		
		    return daoPageResult;
		}
		
		
		/**
		 * 将Campaign BO转为VO
		 * 
		 * @param campaigns
		 * @return
		 */
		private DaoPageResult<CampaignListRowVo> convertCampaignsToVos(
		        DaoPageResult<Campaign> campaigns) {
		    DaoPageResult<CampaignListRowVo> campaignListRowPageResult = ServiceUtil
		            .getResult(campaigns,
		                    new DataTransfer<Campaign, CampaignListRowVo>() {
		
		                        @Override
		                        public CampaignListRowVo transfer(Campaign input) {
		                            CampaignListRowVo campaignListRowVo = new CampaignListRowVo(
		                                    input);
		                            campaignListRowVo.setAdOwnerName(userAcctMgr
		                                    .getUserName(input.getAdOwnerUcid())
		                                    .getUcname());
		                            // 未投放原因
		                            campaignListRowVo
		                                    .setAbnormalMessage(CampaignNoProcessCode
		                                            .fromInt(input.getAbnormalId())
		                                            .getContent());
		                            return campaignListRowVo;
		                        }
		
		                    });
		
		    return campaignListRowPageResult;
		}

### 业务处理： 全局异常检查方案  ###

ExceptionHandler

1. 业务方的 @Valid @PartFormValid throw FieldException 等信息，均会捕捉，并自动拼装返回。
2. 开发者只关注 业务数据本身。

### 排查方案： 全局SessionId ###

SessionInterceptor

1. 系统为每个Session生成UUID
2. slf4j 自动获取此ID
3. 并会将此ID返回给前端，排查方便。

### 业务处理： 注解式Cache方案 ###

UserAcctMgrImpl

### UT： 内存数据库测试方案 ###

BaseTestCase

	datasource_location=jdbc:h2:mem:dsp;MODE=MYSQL;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false;DB_CLOSE_ON_EXIT=FALSE;
	driver_class_name=org.h2.Driver
	username=
	password=
	
	@Before
	public void setUp() {
	
	    try {
	        execute("sql/testdata/demo/demo.sql");
	    } catch (IOException e) {
	        e.printStackTrace();
	    }
	}

## 自惭 ##

1. 使用了@SessionValue进行参数传递，这并不规范。
