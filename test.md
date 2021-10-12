MQTT的重新订阅

```c#
using Dept9IOT.DeviceInfo.IRepository.MongoDB;
using Dept9IOT.DeviceInfo.Model.DbModels;
using Dept9IOT.DeviceInfo.Model.Enums;
using Dept9IOT.DeviceInfo.Model.MongoDBModels;
using Dept9IOT.DeviceInfo.Unity.Consts;
using Dept9IOT.DeviceInfo.Unity.Enums.Mqtt;
using Dept9IOT.DeviceInfo.Unity.Models.BllModels;
using Dept9IOT.DeviceInfo.Unity.Models.MqttDto.Pulish;
using Dept9IOT.DeviceInfo.Unity.MqttDto;
using Dept9IOT.DeviceInfo.WebApi.CoreBuilder.CoreMqtt;
using evoc9.Json;
using evoc9.Redis;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using MQTTnet;
using MQTTnet.Client;
using MQTTnet.Client.Connecting;
using MQTTnet.Client.Disconnecting;
using MQTTnet.Client.Options;
using MQTTnet.Client.Publishing;
using MQTTnet.Protocol;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

namespace Dept9IOT.DeviceInfo.Services.MqttClientService
{
    /// <summary>
    /// MQTT客户端服务
    /// </summary>
    public class NMqttClientService : INMqttClientService
    {
        private readonly ILogger _logger;
        private IMqttClient mqttClient;
        private IMqttClientOptions _options;
        private readonly IServiceScopeFactory _serviceScopeFactory;

        public NMqttClientService(ILogger<NMqttClientService> logger, 
            IMqttClientOptions options,
            IServiceScopeFactory serviceScopeFactory)
        {
            _logger = logger;
            _options = options;
            mqttClient = new MqttFactory().CreateMqttClient();
            ConfigureMqttClient();
            _serviceScopeFactory = serviceScopeFactory;
        }

        private void ConfigureMqttClient()
        {
            mqttClient.ConnectedHandler = this;
            mqttClient.DisconnectedHandler = this;
            mqttClient.ApplicationMessageReceivedHandler = this;
            StartAsync(CancellationToken.None).Wait();
        }

        #region 客户端操作函数
        //连接成功处理函数
        public async Task HandleConnectedAsync(MqttClientConnectedEventArgs eventArgs)
        {
            await Task.CompletedTask;
        }

        //断开连接处理函数
        public async Task HandleDisconnectedAsync(MqttClientDisconnectedEventArgs eventArgs)
        {
            try
            {
                _logger.LogInformation($"mqtt断开连接，正在尝试重连... ...");
                await Task.Delay(TimeSpan.FromSeconds(5));
                await mqttClient.ConnectAsync(_options, CancellationToken.None);
            }
            catch (Exception ex)
            {
                _logger.LogError($"mqtt重联失败，异常信息：{eventArgs.Exception.Message}");
            }
        }

        //启动
        public async Task StartAsync(CancellationToken cancellationToken)
        {
            await mqttClient.ConnectAsync(_options);
            Console.WriteLine("StartAsync:::" + _options.ChannelOptions.ToJsonString());
            if (!mqttClient.IsConnected)
            {
                await mqttClient.ReconnectAsync();
            }
        }

        //停止
        public async Task StopAsync(CancellationToken cancellationToken)
        {
            if (mqttClient.IsConnected)
            {
                if (cancellationToken.IsCancellationRequested)
                {
                    var disconnectOption = new MqttClientDisconnectOptions
                    {
                        ReasonCode = MqttClientDisconnectReason.NormalDisconnection,
                        ReasonString = "NormalDiconnection"
                    };
                    await mqttClient.DisconnectAsync(disconnectOption, cancellationToken);
                }
                await mqttClient.DisconnectAsync();
            }
        }

        //启动，未处理
        public void Start(CancellationToken cancellationToken)
        {
            throw new NotImplementedException();
        }

        //停止，未处理
        public void Stop(CancellationToken cancellationToken)
        {
            throw new NotImplementedException();
        }

        #endregion

        /// <summary>
        /// 订阅主题
        /// </summary>
        /// <param name="topic"></param>
        /// <returns></returns>
        public async Task<bool> SubscribeAsync(string topic)
        {
            try
            {
                await mqttClient.SubscribeAsync(topic, MqttQualityOfServiceLevel.ExactlyOnce);
                return true;
            }
            catch (Exception ex)
            {
                _logger.LogError($"主题：{topic} 订阅失败，错误详情：{ex.Message}",ex);
                throw;
            }
        }

        /// <summary>
        /// 发布消息
        /// </summary>
        /// <param name="topic"></param>
        /// <param name="reploy"></param>
        /// <returns></returns>
        public async Task<bool> PublishAsync(string topic, string reploy)
        {
            var pushResult = await mqttClient.PublishAsync(topic, reploy, MqttQualityOfServiceLevel.ExactlyOnce);
            if (pushResult.ReasonCode == MqttClientPublishReasonCode.Success)
                return true;
            return false;
        }

        /// <summary>
        /// 订阅消息处理函数
        /// </summary>
        /// <param name="eventArgs"></param>
        /// <returns></returns>
        public async Task HandleApplicationMessageReceivedAsync(MqttApplicationMessageReceivedEventArgs eventArgs)
        {
            var operateTypeStr = string.Empty;
            try
            {
                _logger.LogInformation($"{DateTime.Now} 接收到主题为：{eventArgs.ApplicationMessage.Topic}的消息，消息内容为：{Encoding.UTF8.GetString(eventArgs.ApplicationMessage.Payload)}");
                var topic = eventArgs.ApplicationMessage.Topic;
                var topicItem = topic.Split("/");
                var joinToMqttWay = (JoinToMqttWay)(int.Parse(topicItem[2]));
                var deviceId = topic.Split("/")[3];
                var msg = Encoding.UTF8.GetString(eventArgs.ApplicationMessage.Payload);
                var messageDto = JsonConvert.DeserializeObject<MqttMessageDto>(msg);
                operateTypeStr = messageDto.Type.ToString();
                if (messageDto.Code == MqttResultStatusCode.Success)
                {
                    switch (messageDto.Type)
                    {
                        case MqttOperateType.DeviceConnectToGateway:
                            //设备连接网关订阅处理函数
                            await DeviceConnectToGatewaySubHandle(topic, messageDto);
                            break;
                        case MqttOperateType.HongStudy:
                            //设备连接网关订阅处理函数
                            await HongStudySubHandle(topic, messageDto);
                            break;
                        case MqttOperateType.DeviceHeartbeat:
                            //设备心跳处理函数
                            await DeviceHeartbeatSubHandle(topic, messageDto);
                        // case MqttOperateType.DeviceHeartbeat:
                        //     //网关心跳处理函数
                        //     await DeviceHeartbeatSubHandle(topic, messageDto);
                        //     break;
                        default:
                            break;
                    }
                }
                else
                {
                    _logger.LogWarning($"设备层业务处理失败，主题信息：{topic}");
                }
                await Task.CompletedTask;
            }
            catch (Exception ex)
            {
                _logger.LogError($"MQTT订阅消息处理函数出现异常，主题为：{eventArgs.ApplicationMessage.Topic},操作类型：{operateTypeStr}",ex);
            }
        }

        #region 私有方法
        /// <summary>
        /// 设备连接网关订阅处理函数
        /// </summary>
        /// <param name="topic"></param>
        /// <param name="messageDto"></param>
        private async Task DeviceConnectToGatewaySubHandle(string topic, MqttMessageDto messageDto)
        {
            var topicItem = topic.Split("/");
            var joinToMqttWay = (JoinToMqttWay)(int.Parse(topicItem[2]));
            using (var scope = _serviceScopeFactory.CreateScope())
            {
                var dbContext = scope.ServiceProvider.GetRequiredService<_91iotdbContext>();
                var redisClient = scope.ServiceProvider.GetRequiredService<RedisClient>();
                var subDto = JsonConvert.DeserializeObject<DeviceCTRGatewayJoinSbuDto>(messageDto.Msg.ToString());
                string key = ApplicationConst.RedisKey.DeviceConnectedToRealGatewayJoinPreFix + subDto.SendBack.Key;
                if (!redisClient.KeyExists(key))
                    _logger.LogError($"设备连接网关订阅处理函数|出现异常|缓冲不存在；主题信息:{topic}");

                var cacheStr = redisClient.StringGet<string>(key);
                var cacheModel = JsonConvert.DeserializeObject<DeviceJoinCacheModel>(cacheStr);

                var deviceEntitys = await dbContext.Devices.Where(a => a.GCPortId == cacheModel.GCPortId && a.IsDelete == false).ToListAsync();
                if (deviceEntitys.Count > 0)
                {
                    var deviceEntity =await dbContext.Devices.FirstOrDefaultAsync(a => a.Id == cacheModel.DeviceId);
                    if(deviceEntity==null)
                        _logger.LogError($"设备连接网关订阅处理函数|出现异常|查询不到新增设备；主题信息:{topic}，新增设备id:{cacheModel.DeviceId}");

                    deviceEntity.JoinStatus = JoinStatus.Joined;
                    deviceEntity.Status =(int)DeviceStatus.Online;
                    dbContext.Devices.Update(deviceEntity);
                    await dbContext.SaveChangesAsync();

                    var deviceIds = deviceEntitys.Select(a => a.Id).ToList();
                    _logger.LogInformation($"设备成功接入到网关，网关id:{deviceEntitys[0].GatewayId},设备id:{string.Join(",", deviceIds)}");
                }
                else
                {
                    _logger.LogWarning($"设备连接网关出现异常，查询不到接入的设备！");
                }
            }
        }

        /// <summary>
        /// 红外学习处理函数
        /// </summary>
        /// <param name="topic"></param>
        /// <param name="messageDto"></param>
        private async Task HongStudySubHandle(string topic, MqttMessageDto messageDto)
        {
            var topicItem = topic.Split("/");
            var joinToMqttWay = (JoinToMqttWay)(int.Parse(topicItem[2]));
            using (var scope = _serviceScopeFactory.CreateScope())
            {
                var subDto = JsonConvert.DeserializeObject<InfraredStudySubDto>(messageDto.Msg.ToString());
                if (string.IsNullOrEmpty(subDto.CmdCode))
                    return;

                var redisClient = scope.ServiceProvider.GetRequiredService<RedisClient>();
                var dbContext = scope.ServiceProvider.GetRequiredService<_91iotdbContext>();
                using (var transaction = dbContext.Database.BeginTransaction())
                {
                    try
                    {
                        string key = ApplicationConst.RedisKey.InfraredStudyPreFix + subDto.SendBack.Key;
                        if (!redisClient.KeyExists(key))
                            throw new Exception("红外学习的缓存key不存在！");

                        var value = await redisClient.StringGetAsync<string>(key);
                        var deviceId = value.Split("|")[0];
                        var modelCmdId = value.Split("|")[1];

                        //将学习码放在缓存中，前端轮训获取
                        string noticKey = ApplicationConst.RedisKey.InfraredStudyNoticePreFix + deviceId;
                        if (!redisClient.KeyExists(noticKey))
                            throw new Exception("红外学习通知的缓存key不存在！");

                        await redisClient.StringSetAsync(noticKey, subDto.CmdCode, TimeSpan.FromSeconds(300));

                        //更新学习结果到数据库中
                        var modelCmd = await dbContext.ModelCmds.FirstOrDefaultAsync(a => a.Id == modelCmdId);
                        if (modelCmd != null)
                        {
                            modelCmd.Code = subDto.CmdCode;
                            dbContext.Update(modelCmd);
                        }

                        lock (this)
                        {
                            //将学习码更新到设备命令表中
                            var deviceCmd = dbContext.DeviceCmds.FirstOrDefault(a => a.Name == modelCmd.Name && a.DeviceId == deviceId);
                            if (deviceCmd == null)
                            {
                                var cmdEntity = new DeviceCmd
                                {
                                    CmdId = Guid.NewGuid().ToString(),
                                    DeviceId = deviceId,
                                    Code = subDto.CmdCode,
                                    CreateBy = modelCmd.UpdateBy,
                                    CreateTime = DateTime.Now,
                                    Name = modelCmd.Name,
                                    UpdateBy = modelCmd.UpdateBy,
                                    UpdateTime = DateTime.Now,
                                    Remark = string.Empty
                                };
                                dbContext.Add(cmdEntity);
                            }
                            else
                            {
                                deviceCmd.Code = subDto.CmdCode;
                                dbContext.Update(deviceCmd);
                            }

                            dbContext.SaveChanges();
                            transaction.Commit();
                            redisClient.KeyDelete(key);
                            redisClient.KeyDelete(noticKey);
                        }
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError($"订阅红外学习结果失败，错误详情：{ex.Message}");
                        await transaction.RollbackAsync();
                        throw;
                    }
                }
            }
        }

        /// <summary>
        /// 设备心跳处理函数
        /// </summary>
        /// <param name="topic"></param>
        /// <param name="messageDto"></param>
        /// <returns></returns>
        private async Task DeviceHeartbeatSubHandle(string topic, MqttMessageDto messageDto)
        {
            var topicItem = topic.Split("/");
            var gatewayId = topicItem[3];
            var subDto = JsonConvert.DeserializeObject<DeviceHeartbeatSubDto>(messageDto.Msg.ToString());
            using (var scope = _serviceScopeFactory.CreateScope())
            {
                //更新设备状态
                var dbContext = scope.ServiceProvider.GetRequiredService<_91iotdbContext>();
                var query = from a in dbContext.GatewayCommunicationPorts
                            join b in dbContext.Devices on a.Id equals b.GCPortId
                            where a.PortCode == subDto.PortNum && a.IsDelete == false && b.GatewayId == gatewayId && b.AddressBitNum == subDto.DeviceAddr && b.IsDelete == false
                            select b;

                var deviceEntitys =await query.ToListAsync();
                if (deviceEntitys.Count == 0)
                {
                    _logger.LogError($"设备心跳上报失败，查找不到设备，设备挂接的网关端口编码:{subDto.PortNum},设备的地址位：{subDto.DeviceAddr},主题信息：{topic}");
                    return;
                }

                var deviceEntity = deviceEntitys.FirstOrDefault() ;
                deviceEntity.Status = (int)subDto.Alive;
                deviceEntity.StatusUpdateTime = DateTime.Now;
                dbContext.Update(deviceEntity);
                await dbContext.SaveChangesAsync();

                //将状态持久化到状态历史表
                var baseRepository = scope.ServiceProvider.GetRequiredService<IDeviceStateHistoryRepository>();
                var history = new DeviceStateHistory
                {
                    Id = Guid.NewGuid().ToString(),
                    DeviceId = deviceEntity.Id,
                    DeviceName = deviceEntity.Name,
                    Status = subDto.Alive,
                    CreateTime = DateTime.Now
                };
                await baseRepository.Insert(history);
            }
        }
        #endregion
    }
}

```

```
using Dept9IOT.DeviceInfo.IRepository;
using Dept9IOT.DeviceInfo.IServices;
using Dept9IOT.DeviceInfo.Model.DbModels;
using Dept9IOT.DeviceInfo.Unity;
using Dept9IOT.DeviceInfo.Unity.DtoModel;
using Dept9IOT.DeviceInfo.Unity.Enums;
using Dept9IOT.DeviceInfo.Unity.Helpers;
using Dept9IOT.DeviceInfo.Unity.InputModel;
using Dept9IOT.DeviceInfo.Unity.InputModel.UpdateInput;
using Dept9IOT.DeviceInfo.Unity.Models.InputModel.DeviceBrand;
using evoc9.Collections;
using evoc9.Common;
using evoc9.Linq;
using evoc9.Redis;
using Evoc9.Common;
using Evoc9.Common.Client;
using Evoc9.EFCore.SqlBase;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using System.Text;
using System.Threading.Tasks;

namespace Dept9IOT.DeviceInfo.Services
{
     public  class DeviceBrandService:IDeviceBrandService
    {
        private readonly ILogger _logger;
        private readonly _91iotdbContext _dbContext;

        public DeviceBrandService(ILogger<DeviceBrandService> logger,_91iotdbContext dbContext)
        {
            _logger = logger;
            _dbContext = dbContext;
    }

        //删除
        public async Task Delete(string id)
        {
            var entity = await _dbContext.DeviceBrands.FirstOrDefaultAsync(a => a.Id == id && a.IsDelete == false);
            if (entity == null)
                throw new Exception("要删除的品牌不存在！");

            entity.IsDelete = true;
           // _dbContext.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        //批量删除
        public async Task Delete(List<string> ids)
        {
            using var tran = _dbContext.Database.BeginTransaction();
            try
            {
                var entitys = await _dbContext.DeviceBrands.Where(a => ids.Contains(a.Id) && a.IsDelete == false).ToListAsync();

                foreach (var item in entitys)
                {
                    item.IsDelete = true;
                }

                _dbContext.UpdateRange(entitys);
                await _dbContext.SaveChangesAsync();

                await tran.CommitAsync();
            }
            catch (Exception ex)
            {
                await tran.RollbackAsync();
                throw new Exception($"删除失败!详情：{ex.Message}");
            }
        }

        //获取
        public async Task<DeviceBrand> Get(string id)
        {
            var entity = await _dbContext.DeviceBrands.FirstOrDefaultAsync(a => a.Id == id && a.IsDelete == false);
            if(entity==null)
                throw new Exception("品牌不存在！");
            return entity;
        }

        //获取分页列表
        public async Task<PageResult<DeviceBrandDto>> GetListBySearch(DeviceBrandSearchInput input)
        {
            var query = from a in _dbContext.DeviceBrands
                        join b in _dbContext.DeviceModels on a.Id equals b.BrandId
                        select new
                        {
                            DeviceBrand = a,
                            DeviceModel = b
                        };

            var queryTuple = await query.ToListAsync();
            List<DeviceBrand> entitys;
            int count;
            count = await _dbContext.DeviceBrands
                .WhereIf(a=> a.Name.Contains(input.KeyWord) || a.EnName.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                .WhereIf(a => a.DictId == input.DictId, !string.IsNullOrEmpty(input.DictId))
                .WhereIf(a => a.BrandCategory == (int)input.BrandCategory, input.BrandCategory != BrandCategory.CheckAll)
                .Where(a =>a.IsDelete==false).CountAsync();
            entitys = await _dbContext.DeviceBrands
                 .WhereIf(a => a.Name.Contains(input.KeyWord) || a.EnName.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                .WhereIf(a => a.DictId == input.DictId, !string.IsNullOrEmpty(input.DictId))
                .WhereIf(a => a.BrandCategory == (int)input.BrandCategory, input.BrandCategory != BrandCategory.CheckAll)
                .Where(a => a.IsDelete == false)
                .Skip((input.PageIndex - 1) * input.PageSize).Take(input.PageSize).ToListAsync();
            PageResult<DeviceBrandDto> pageResult = new PageResult<DeviceBrandDto>();
            List<DeviceBrandDto> dtos = new List<DeviceBrandDto>();
            
            entitys.ForEach(a =>
            {
                DeviceBrandDto dto = new DeviceBrandDto();
                //同等效果代码：
                dto = ExpressionMapper<DeviceBrand, DeviceBrandDto>.Map(a);
                dto.BrandCategory = (BrandCategory)a.BrandCategory;
                dto.Number = queryTuple.Where(b => b.DeviceModel.BrandId == a.Id && b.DeviceModel.IsDelete == false).Count();
                dtos.Add(dto);
            });

            pageResult.DataList = dtos;
            pageResult.PageIndex = input.PageIndex;
            pageResult.PageSize = input.PageSize;
            pageResult.TotalCount = count;
            return pageResult;
        }

        private DeviceBrandSimpleDto GetModelDto(DeviceBrand deviceBrand)
        {
            return new DeviceBrandSimpleDto
            {
                //BrandCategory = (BrandCategory)deviceBrand.BrandCategory,
                DictId = deviceBrand.DictId,
                Id = deviceBrand.Id,
                Name = deviceBrand.Name,
                EnName = deviceBrand.EnName
            };
        }

        //获取简单数据列表
        public async Task<List<DeviceBrandSimpleDto>> GetSimpleList()
        {
            var deviceBrandList = await _dbContext.DeviceBrands.Where(a => a.IsDelete != true).ToListAsync();
            List<DeviceBrandSimpleDto> list = null;
            if(deviceBrandList != null)
            {
                list = new List<DeviceBrandSimpleDto>();
                deviceBrandList.ForEach(a =>
                {
                    
                    list.Add(GetModelDto(a));
                });
            }
            return list;
        }

        //根据DictId获取品牌简单数据列表
        public async Task<List<DeviceBrandSimpleDto>> GetSimpleListByDictId(string dictId)
        {
            var deviceBrandList = await _dbContext.DeviceBrands.Where(a =>a.DictId==dictId && a.IsDelete == false).ToListAsync();
            List<DeviceBrandSimpleDto> list = null;
            if (deviceBrandList != null)
            {
                list = new List<DeviceBrandSimpleDto>();
                deviceBrandList.ForEach(a =>
                {
                    
                    list.Add(GetModelDto(a));
                });
            }
            return list;
        }

        //插入
        public async Task Insert(DeviceBrand deviceBrand)
        {
            //先从数据库里查出来 查重
            var entity = await _dbContext.DeviceBrands.FirstOrDefaultAsync(a => a.Name == deviceBrand.Name && a.DictId == deviceBrand.DictId && a.IsDelete == false);
            if (entity != null)
                throw new Exception("品牌已存在，请勿重复添加！");

            await _dbContext.DeviceBrands.AddAsync(deviceBrand);
            await _dbContext.SaveChangesAsync();
        }

        //更新
        public async Task Update(string _userId, DeviceBrandUpdateInput input)
        {
            //先从数据库里查出来，逐个改状态，除了唯一ID和Creat系列都改变
            var entity = await _dbContext.DeviceBrands.FirstOrDefaultAsync(a => a.Id == input.Id && a.IsDelete == false);
            if (entity == null)
            {
                throw new Exception("要更新的品牌ID不存在");
            }
            {
                entity.BrandCategory = (int)input.BrandCategory;
                entity.DictId = input.DictId;
                entity.Name = input.Name;
                entity.EnName = input.EnName;
                entity.Remark = input.Remark;
                entity.UpdateBy = _userId;
                entity.UpdateTime = DateTime.Now;
            }
            _dbContext.DeviceBrands.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        #region A06版本

        //根据品牌类型获取品牌简单数据列表
        public async Task<List<DeviceBrandSimpleDto>> GetSimpleListByGatewayCategory(BrandCategory brandCategory)
        {
            var deviceBrandList = await _dbContext.DeviceBrands
                 .WhereIf(a => a.BrandCategory == (int)brandCategory, brandCategory != BrandCategory.CheckAll)
                 .Where(a => a.IsDelete == false).ToListAsync();
               
            List<DeviceBrandSimpleDto> list = null;
            if (deviceBrandList != null)
            {
                list = new List<DeviceBrandSimpleDto>();
                deviceBrandList.ForEach(a =>
                {
                    
                    list.Add(GetModelDto(a));
                });
            }
            return list;
        }

        

        public Task<List<DeviceBrandSimpleDto>> GetSimInfoListBySearch(BrandCategory brandCategory)
        {
            throw new NotImplementedException();
        }
        #endregion
    }
}

```

```c#
using Evoc9.Common;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Dept9IOT.DeviceInfo.Unity.InputModel;
using Dept9IOT.DeviceInfo.IServices;
using Evoc9.Common.Aop;
using Dept9IOT.DeviceInfo.Model.DbModels;
using evoc9.Filter;
using System.ComponentModel.DataAnnotations;
using Dept9IOT.DeviceInfo.Unity.InputModel.UpdateInput;
using Dept9IOT.DeviceInfo.Unity.DtoModel;
using Dept9IOT.DeviceInfo.Unity.Enums;
using Dept9IOT.DeviceInfo.Unity.Models.InputModel.DeviceBrand;

namespace Dept9IOT.DeviceInfo.WebApi.Controllers.A05
{
    /// <summary>
    /// 品牌表的增删改查接口
    /// </summary>
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class DeviceBrandController : ControllerBase
    {
        private readonly ILogger<DeviceBrandController> _logger;
        private readonly IDeviceBrandService _deviceBrandService;

        public DeviceBrandController(ILogger<DeviceBrandController> logger, IDeviceBrandService deviceBrandService)
        {
            _logger = logger;
            _deviceBrandService = deviceBrandService;
        }

        /// <summary>
        /// 插入一条品牌分类（后台对接）
        /// </summary>
        /// <param name="input">品牌模型输入</param>
        /// <returns></returns>
        [HttpPost("A06/Add")]
        public async Task<ApiResult> Insert(DeviceBrandInput input)
        {
            var _userId = User.GetLoginUserId();
            var verify = InputVerifyExtension.Verify(input);
            if (!string.IsNullOrEmpty(verify))
            {
                return ApiResultUnity.Error(ApiEnum.Error, $"校验失败：{verify}");
            }
            DeviceBrand deviceBrand = new DeviceBrand
            {
                DictId = input.DictId,
                BrandCategory = (int)input.BrandCategory,
                Name = input.Name,
                EnName = input.EnName,
                Remark = input.Remark,
                Id = Guid.NewGuid().ToString(),
                CreateBy = _userId,
                CreateTime = DateTime.Now,
                UpdateBy = _userId,
                UpdateTime = DateTime.Now,
            };

            await _deviceBrandService.Insert(deviceBrand);
            return ApiResultUnity.Success();
        }

        /// <summary>
        /// 根据品牌id获取一条信息（后台对接）
        /// </summary>
        /// <param name="id">品牌ID</param>
        /// <returns></returns>
        [HttpGet("A06/Get/{id}")]
        [ProducesResponseType(typeof(DeviceBrand), 200)]
        public async Task<ApiResult> Get([Required] string id)
        {
            var data = await _deviceBrandService.Get(id);
            return ApiResultUnity.Success(data);
        }

        /// <summary>
        /// 根据id删除一条品牌信息（后台对接）
        /// </summary>
        /// <param name="id">要删除的品牌ID</param>
        /// <returns></returns>
        [HttpDelete("A06/Delete/{id}")]
        public async Task<ApiResult> Delete([Required] string id)
        {

            await _deviceBrandService.Delete(id);
            return ApiResultUnity.Success();
        }

        /// <summary>
        /// 根据ids列表删除（后台对接）
        /// </summary>
        /// <param name="ids">要删除的ids</param>
        /// <returns></returns>
        [HttpDelete("A06/Delete/List")]
        public async Task<ApiResult> Delete([FromBody] List<string> ids)
        {
            await _deviceBrandService.Delete(ids);
            return ApiResultUnity.Success();
        }

        /*/// <summary>
        /// 获取品牌简单数据列表（APP 对接)
        /// </summary>
        /// <returns></returns>
        [HttpGet("A05/List")]
        [ProducesResponseType(typeof(List<DeviceBrandSimpleDto>), 200)]
        public async Task<ApiResult> GetSimpleList()
        {
            var data = await _deviceBrandService.GetSimpleList();
            return ApiResultUnity.Success(data);
        }*/

        /// <summary>
        /// 根据数据字典id(品类Id)获取品牌简单数据列表（APP对接）
        /// </summary>
        /// <param name="dictId">数据字典id(品类Id)</param>
        /// <returns></returns>
        [HttpGet("A06/SimInfo/List/ByDictId")]
        [ProducesResponseType(typeof(List<DeviceBrandSimpleDto>), 200)]
        public async Task<ApiResult> GetSimpleListByDictId(string dictId)
        {
            var data = await _deviceBrandService.GetSimpleListByDictId(dictId);
            return ApiResultUnity.Success(data);
        }


       
        /// <summary>
        /// 分页查询
        /// </summary>
        /// <param name="input"></param>
        /// <returns></returns>
        [HttpGet("A06/Search/List")]
        [ProducesResponseType(typeof(PageResult<DeviceBrandDto>), 200)]
        public async Task<ApiResult> GetListBySearch([FromQuery]DeviceBrandSearchInput input)
        {
            var data = await _deviceBrandService.GetListBySearch(input);
            return ApiResultUnity.Success(data);
        }


        /// <summary>
        /// 更新数据（后台对接）
        /// </summary>
        /// <param name="input">品牌更新模型</param>
        /// <returns></returns>
        [HttpPut("A06/Update")]
        public async Task<ApiResult> Update([FromBody] DeviceBrandUpdateInput input)
        {
            var _userId = User.GetLoginUserId();

            await _deviceBrandService.Update(_userId, input);
            return ApiResultUnity.Success();
        }

        /// <summary>
        /// 根据品牌类型获取品牌简单数据列表（APP对接）
        /// </summary>
        /// <param name="brandCategory">品牌类型（0全部 1设备 2网关）</param>
        /// <returns></returns>
        [HttpGet("A06/SimInfo/List/ByBrandCategory")]
        [ProducesResponseType(typeof(List<DeviceBrandSimpleDto>), 200)]
        public async Task<ApiResult> GetSimpleListByGatewayCategory([Required]BrandCategory brandCategory)
        {
            var data = await _deviceBrandService.GetSimpleListByGatewayCategory(brandCategory);
            return ApiResultUnity.Success(data);
        }

    }   
}
```

```c#
using Dept9IOT.DeviceInfo.IRepository.Base;
using Dept9IOT.DeviceInfo.Model.DbModels;
using Dept9IOT.DeviceInfo.Unity.DtoModel;
using Dept9IOT.DeviceInfo.Unity.Enums;
using Dept9IOT.DeviceInfo.Unity.InputModel;
using Dept9IOT.DeviceInfo.Unity.InputModel.UpdateInput;
using Dept9IOT.DeviceInfo.Unity.Models.InputModel.DeviceBrand;
using Evoc9.Common;
using Evoc9.EFCore.SqlBase;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;

namespace Dept9IOT.DeviceInfo.IServices
{
    public interface IDeviceBrandService
    {
        /// <summary>
        /// 插入
        /// </summary>
        /// <param name="entity"></param>
        /// <returns></returns>
        Task Insert(DeviceBrand entity);

        /// <summary>
        /// 更新
        /// </summary>
        /// <param name="_userId"></param>
        /// <param name="input"></param>
        /// <returns></returns>
        Task Update( string _userId, DeviceBrandUpdateInput input);

        /// <summary>
        /// 根据id获取
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        Task<DeviceBrand> Get(string id);

        /// <summary>
        /// 根据id删除
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        Task Delete(string id);

        /// <summary>
        /// 根据id列表删除
        /// </summary>
        /// <param name="ids"></param>
        /// <returns></returns>
        Task Delete(List<string> ids);

        /// <summary>
        /// 分页查询
        /// </summary>
        /// <param name="input"></param>
        /// <returns></returns>
        Task<PageResult<DeviceBrandDto>> GetListBySearch(DeviceBrandSearchInput input);

        /// <summary>
        /// 获取品牌简单数据列表
        /// </summary>
        /// <returns></returns>
        Task<List<DeviceBrandSimpleDto>> GetSimpleList();

        /// <summary>
        /// 根据DictId获取品牌简单数据列表
        /// </summary>
        /// <param name="dictId">数据字典Id(品类Id)</param>
        /// <returns></returns>
        Task<List<DeviceBrandSimpleDto>> GetSimpleListByDictId(string dictId);



        #region A06 版本
        /// <summary>
        /// 根据品牌类型获取品牌简单数据列表
        /// </summary>
        /// <param name="gatewayCategory">品牌类型</param>
        /// <returns></returns>
        Task<List<DeviceBrandSimpleDto>> GetSimpleListByGatewayCategory(BrandCategory brandCategory);

        
        #endregion
    }
}

```

```
using CrateProjectModel.Models;
using Dept9IOT.DeviceInfo.IServices;
using Dept9IOT.DeviceInfo.Model.DbModels;
using Dept9IOT.DeviceInfo.Unity.DtoModel.GatewayDevice;
using Dept9IOT.DeviceInfo.Unity.InputModel.CustomProtocol;
using Evoc9.EFCore.SqlBase;
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading.Tasks;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using evoc9.Collections;
using Dept9IOT.DeviceInfo.Model.Enums;
using Dept9IOT.DeviceInfo.Unity.Helpers;
using Dept9IOT.DeviceInfo.Unity.InputModel.Gateway;
using Dept9IOT.DeviceInfo.WebApi.CoreBuilder.CoreMqtt;
using Dept9IOT.DeviceInfo.Unity.Consts;
using Dept9IOT.DeviceInfo.Unity.Models.InputModel.Gateway;

namespace Dept9IOT.DeviceInfo.Services
{
    /// <summary>
    /// 网关服务
    /// </summary>
    public class GatewayService : IGatewayService
    {
        private readonly _91iotdbContext _dbContext;
        private readonly INMqttClientService _mqttClientService;

        public GatewayService(_91iotdbContext dbContext,
            INMqttClientService mqttClientService)
        {
            _dbContext = dbContext;
            _mqttClientService = mqttClientService;
        }

        //添加
        public async Task Insert(Gateway entity)
        {
            var t_entity =await _dbContext.Gateways.FirstOrDefaultAsync(a=>(a.Snid==entity.Snid||a.Mac==entity.Mac)&&a.IsDelete==false);
            if (t_entity != null)
                throw new Exception($"添加失败，已存在Snid为{entity.Snid}，或Mac为{entity.Mac}的网关");
            await _dbContext.AddAsync(entity);
            await _dbContext.SaveChangesAsync();
        }

        //更新网关信息
        public async Task UpdateByDto(UpdateGatewayInput input)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a => a.Id == input.Id);
            if (entity == null)
                throw new Exception("要删除的网关不存在");
            entity.Mac = input.Mac;
            entity.Ip = input.Ip;
            entity.Mac = input.Mac;
            entity.Snid = input.Snid;
            entity.Name = input.Name;
            entity.QrCodeInfo = input.QrCodeInfo;
            entity.BrandId = input.BrandId;
            entity.ModelId = input.ModelId;
            entity.Remark = input.Remark;
            _dbContext.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        //根据id删除
        public async Task Delete(string Id)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a=>a.Id== Id);
            if (entity == null)
                throw new Exception("要删除的网关不存在");
            entity.IsDelete = true;
            _dbContext.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        //接入真实网关
        public async Task<GatewayBriefDto> JoinRealGateway(JoinRealGatewayInput input)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a => a.Snid== input.Snid&&a.IsDelete==false);
            if (entity == null)
                throw new Exception($"接入网关失败，网关尚未录入系统，网关snid:{input.Snid}!");
            entity.Mac = input.Mac;
            entity.Ip = input.Ip;
            entity.JoinStatus = JoinStatus.Joined;
            entity.Status = DeviceStatus.Offline;
            await _dbContext.SaveChangesAsync();

            //订阅服务
            var resultTopic = ApplicationConst.SubToticPrefix+ $"{(int)JoinToMqttWay.Real}/{entity.Id}";
            var subscribeResult = await _mqttClientService.SubscribeAsync(resultTopic);
            return new GatewayBriefDto { Id= entity.Id, GwHbTime = entity.StatusTimeInterval};
        }

        /// <summary>
        /// 检查接入状态
        /// </summary>
        /// <param name="sind"></param>
        /// <returns></returns>
        public async Task CheckJoinStatus(string sind, string userId)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a => a.Snid == sind);
            if (entity == null)
                throw new Exception("网关未接入！");
            entity.UserId = userId;
            _dbContext.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        //接入虚拟网关
        public async Task<GatewayBriefDto> JoinVirtualGateway(JoinVirtualGatewayInput input)
        {
            using var tran = _dbContext.Database.BeginTransaction();
            try
            {
                var verifyCode = await _dbContext.VerifyCodes.FirstOrDefaultAsync(a => a.Code == input.VerifyCode);
                if (verifyCode == null || verifyCode?.CreateTime.AddSeconds((double)verifyCode?.ValidTime) > DateTime.Now || verifyCode.IsValid == false)
                    throw new Exception("验证码不存在!");

                verifyCode.IsValid = false;
                _dbContext.Update(verifyCode);
                Gateway entity = new Gateway
                {
                    Id = Guid.NewGuid().ToString(),
                    Mac = input.Mac,
                    UserId = verifyCode.UserId,
                    Ip = input.Ip,
                    Snid = input.Sind,
                    Remark = input.Remark,
                    JoinStatus = JoinStatus.Joined,
                    Status = DeviceStatus.Online
                };

                await _dbContext.AddAsync(entity);
                await tran.CommitAsync();
                return new GatewayBriefDto { Id = entity.Id, GwHbTime = entity.StatusTimeInterval };
            }
            catch (Exception ex)
            {
                await tran.RollbackAsync();
                throw new Exception($"接入失败!详情：{ex.Message}");
            }
        }

        //根据id获取信息
        public async Task<Gateway> Get(string Id)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a => a.Id == Id);
            if (entity == null)
                throw new Exception("查询的网关不存在");

            return entity;
        }

        //根据id获取dto信息
        public async Task<GatewayDto> GetDto(string Id)
        {
            var entity = await _dbContext.Gateways.FirstOrDefaultAsync(a => a.Id == Id);
            if (entity == null)
                throw new Exception("查询的网关不存在");

            var brandEntity = await _dbContext.DeviceBrands.FirstOrDefaultAsync(a => a.Id == entity.BrandId);
            var modelEntity = await _dbContext.DeviceModels.FirstOrDefaultAsync(a => a.Id == entity.ModelId);

            var dto = ExpressionMapper<Gateway, GatewayDto>.Map(entity);
            dto.BrandName = brandEntity != null ? brandEntity.Name : string.Empty;
            dto.ModelName = modelEntity != null ? modelEntity.Name : string.Empty;
            return dto;
        }

        //搜索获取分页信息列表
        public async Task<PageResult<GatewayDto>> GetListBySearch(GatewaySearchInput input)
        {
            var entitys =await _dbContext.Gateways
                .WhereIf(a => a.Name.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                .WhereIf(b => b.GatewayType == input.GatewayType, input.GatewayType != GatewayType.CheckAll)
                .WhereIf(b => b.Status == input.Status, input.Status != DeviceStatus.CheckAll)
                .WhereIf(b => b.JoinStatus == input.JoinStatus, input.JoinStatus != JoinStatus.CheckAll)
                .WhereIf(b => b.GatewayType == input.GatewayType, input.GatewayType != GatewayType.CheckAll)
                .Where(c=>c.IsDelete==false)
                .Skip((input.PageIndex - 1) * input.PageSize).Take(input.PageSize).ToListAsync();

            var count = await _dbContext.Gateways
               .WhereIf(a => a.Name.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
               .WhereIf(b => b.Status == input.Status, input.Status != DeviceStatus.CheckAll)
               .WhereIf(b => b.JoinStatus == input.JoinStatus, input.JoinStatus != JoinStatus.CheckAll)
               .WhereIf(b => b.GatewayType == input.GatewayType, input.GatewayType != GatewayType.CheckAll)
               .Where(c => c.IsDelete == false).CountAsync();

            var brandIds = entitys.Select(a=>a.BrandId).Distinct().ToList();
            var brandEntitys = await _dbContext.DeviceBrands.Where(a => brandIds.Contains(a.Id)).ToListAsync();

            var modelIds = entitys.Select(a => a.ModelId).Distinct().ToList();
            var modelIdEntitys = await _dbContext.DeviceModels.Where(a => modelIds.Contains(a.Id)).ToListAsync();

            PageResult<GatewayDto> pageResult = new PageResult<GatewayDto>();
            List<GatewayDto> dtos = new List<GatewayDto>();
            entitys.ForEach(item =>
            {
                GatewayDto dto = new GatewayDto();
                dto = ExpressionMapper<Gateway, GatewayDto>.Map(item);
                var brand = brandEntitys.FirstOrDefault(a=>a.Id==item.BrandId);
                var model = modelIdEntitys.FirstOrDefault(a => a.Id == item.ModelId);
                dto.BrandName = brand != null ? brand.Name : string.Empty;
                dto.ModelName = model != null ? model.Name : string.Empty;
                dtos.Add(dto);
            });

            pageResult.DataList = dtos;
            pageResult.PageIndex = input.PageIndex;
            pageResult.PageSize = input.PageSize;
            pageResult.TotalCount = count;
            return pageResult;
        }
    }
}

```

```
using CrateProjectModel.Models;
using Dept9IOT.DeviceInfo.IServices;
using Dept9IOT.DeviceInfo.Model.DbModels;
using Dept9IOT.DeviceInfo.Model.Enums;
using Dept9IOT.DeviceInfo.Unity.DtoModel.Device;
using Dept9IOT.DeviceInfo.Unity.DtoModel.GatewayCommunicationPort;
using Dept9IOT.DeviceInfo.Unity.Helpers;
using Dept9IOT.DeviceInfo.Unity.InputModel.Device;
using Dept9IOT.DeviceInfo.Unity.InputModel.GatewayCommunicationPort;
using evoc9.Collections;
using Evoc9.EFCore.SqlBase;
using Microsoft.EntityFrameworkCore;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Dept9IOT.DeviceInfo.Services
{
    /// <summary>
    /// 网关通讯端口服务接口
    /// </summary>
    public class GatewayCommunicationPortService : IGatewayCommunicationPortService
    {
        private readonly _91iotdbContext _dbContext;

        public GatewayCommunicationPortService(_91iotdbContext dbContext)
        {
            _dbContext = dbContext;
        }

        //插入
        public async Task Insert(GatewayCommunicationPort entity)
        {
            await _dbContext.AddAsync(entity);
            await _dbContext.SaveChangesAsync();
        }

        //根据id删除
        public async Task Delete(string Id)
        {
            var entity =await _dbContext.GatewayCommunicationPorts.FirstOrDefaultAsync(a=>a.Id == Id);
            if (entity == null)
                throw new Exception("网关端口不存在！");
            entity.IsDelete = true;
            _dbContext.Update(entity);
            await _dbContext.SaveChangesAsync();
        }

        //根据id获取
        public Task<GatewayCommunicationPort> Get(string Id)
        {
            throw new NotImplementedException();
        }

        //获取通讯端口信息
        public async Task<GatewayCommunicationPortDto> GetDto(string Id)
        {
            var entity = await _dbContext.GatewayCommunicationPorts.FirstOrDefaultAsync(a => a.Id == Id&&a.IsDelete==false);
            if(entity==null)
                throw new Exception("查询的设备不存在！");

            var gatewayEntity = await _dbContext.Gateways.FirstOrDefaultAsync(a=>a.Id== entity.GatewayId&&a.IsDelete==false);
            if(gatewayEntity==null)
                throw new Exception("通讯端口所属的网关不存在，请检查数据的完整性！");

            var dto = ExpressionMapper<GatewayCommunicationPort, GatewayCommunicationPortDto>.Map(entity);
            dto.GatewayName = gatewayEntity.Name;
            return dto;
        }

        //搜索获取分页信息列表
        public async Task<PageResult<GatewayCommunicationPortDto>> GetListBySearch(GCPortSearchInput input)
        {
            var entitys = await _dbContext.GatewayCommunicationPorts
                .WhereIf(a=>a.GatewayId== input.GatewayId,!string.IsNullOrEmpty(input.GatewayId))
                    .WhereIf(a => a.PortName.Contains(input.KeyWord) || a.GatewayId.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                    .WhereIf(b => b.CommunicationModeType == input.PortType, input.PortType != CommunicationModeType.CheckAll)
                    .Where(f => f.IsDelete == false)
                    .Skip((input.PageIndex - 1) * input.PageSize).Take(input.PageSize).ToListAsync();

            var count = await _dbContext.GatewayCommunicationPorts
                .WhereIf(a => a.GatewayId == input.GatewayId, !string.IsNullOrEmpty(input.GatewayId))
                 .WhereIf(a => a.PortName.Contains(input.KeyWord) || a.GatewayId.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                 .WhereIf(b => b.CommunicationModeType == input.PortType, input.PortType != CommunicationModeType.CheckAll)
                .Where(f => f.IsDelete == false).CountAsync();

            var gatewayIds = entitys.Select(a=>a.GatewayId).ToList();
            var gatewagyEntitys = await _dbContext.Gateways.Where(a => gatewayIds.Contains(a.Id) && a.IsDelete == false).ToListAsync();

            PageResult<GatewayCommunicationPortDto> pageResult = new PageResult<GatewayCommunicationPortDto>();
            List<GatewayCommunicationPortDto> dtos = new List<GatewayCommunicationPortDto>();
            entitys.ForEach(item =>
            {       
                var dto = ExpressionMapper<GatewayCommunicationPort, GatewayCommunicationPortDto>.Map(item);
                var gateway = gatewagyEntitys.FirstOrDefault(a => a.Id == item.GatewayId);
                dto.GatewayName = gateway != null ? gateway.Name : string.Empty;
                dtos.Add(dto);
            });

            pageResult.DataList = dtos;
            pageResult.PageIndex = input.PageIndex;
            pageResult.PageSize = input.PageSize;
            pageResult.TotalCount = count;
            return pageResult;

        }

        //搜索获取简要信息分页列表
        public async Task<PageResult<GCPortSimDto>> GetSimInfoListBySearch(GCPortSearchInput input)
        {
            var entitys = await _dbContext.GatewayCommunicationPorts
                .WhereIf(a => a.GatewayId == input.GatewayId, !string.IsNullOrEmpty(input.GatewayId))
                        .WhereIf(a => a.PortName.Contains(input.KeyWord) || a.GatewayId.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                        .WhereIf(b => b.CommunicationModeType == input.PortType, input.PortType != CommunicationModeType.CheckAll)
                        .Where(f => f.IsDelete == false)
                        .Skip((input.PageIndex - 1) * input.PageSize).Take(input.PageSize).ToListAsync();

            var count = await _dbContext.GatewayCommunicationPorts
                .WhereIf(a => a.GatewayId == input.GatewayId, !string.IsNullOrEmpty(input.GatewayId))
                 .WhereIf(a => a.PortName.Contains(input.KeyWord) || a.GatewayId.Contains(input.KeyWord), !string.IsNullOrEmpty(input.KeyWord))
                 .WhereIf(b => b.CommunicationModeType == input.PortType, input.PortType != CommunicationModeType.CheckAll)
                .Where(f => f.IsDelete == false).CountAsync();

            PageResult<GCPortSimDto> pageResult = new PageResult<GCPortSimDto>();
            List<GCPortSimDto> dtos = new List<GCPortSimDto>();
            entitys.ForEach(item =>
            {
                var dto = ExpressionMapper<GatewayCommunicationPort, GCPortSimDto>.Map(item);
                dtos.Add(dto);
            });

            pageResult.DataList = dtos;
            pageResult.PageIndex = input.PageIndex;
            pageResult.PageSize = input.PageSize;
            pageResult.TotalCount = count;
            return pageResult;
        }
    }
}

```

