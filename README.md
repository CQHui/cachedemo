# springboot 缓存demo

## 项目简介
项目为springboot中有关于spring-data的缓存demo
其中持久层用的是mybatis框架+tk插件

个人感觉tk插件比generate更加友好（省去了生成出一大堆看着糟心的代码）
## 项目中运用缓存

### 需求

1. 在查询数据的时候设置缓存 
2. 更新数据的时候同步缓存
3. 删除数据的时候删掉缓存
4. 应用进程重启时初始化缓存

### 实现方式

- 哪里需要写哪里。怎么写，redisTemplate不会用吗 
- 切面编程了解下，易用易扩展
- 我选择注解驱动的Spring cache 

redis配置和cacheManager配置  

---

``` java
@EnableCaching
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
public class ReidsConfig extends CachingConfigurerSupport {
    @Bean(name = "redisTemplate")
    @SuppressWarnings("unchecked")
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {

        RedisTemplate<Object, Object> template = new RedisTemplate<>();

        //使用fastjson序列化
        FastJsonRedisSerializer fastJsonRedisSerializer = new FastJsonRedisSerializer(Object.class);
        // value值的序列化采用fastJsonRedisSerializer
        template.setValueSerializer(fastJsonRedisSerializer);
        template.setHashValueSerializer(fastJsonRedisSerializer);
        // key的序列化采用StringRedisSerializer
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        template.setConnectionFactory(redisConnectionFactory);

        return template;
    }


    /**
     *  缓存管理器
     */
    @Bean
    public CacheManager cacheManager(RedisTemplate<Object, Object> redisTemplate) {
        RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate);
        // 设置默认过期时间为1小时
        cacheManager.setDefaultExpiration(3600);
        return cacheManager;
    }

}
```


服务层注解

``` java
@Service
public class SysUserServiceImpl implements SysUserService{
    @Resource
    private SysUserMapper userMapper;

    @Override
    public List<SysUser> getUsers() {
        return userMapper.selectAll();
    }

    @Override
    @Cacheable(value = "user", key = "'user_'+#userId")
    public SysUser getUserById(Long userId) {
        return userMapper.selectByPrimaryKey(userId);
    }

    @Override
    @CachePut(value = "user", key="'user_'+#sysUser.id")
    public SysUser saveUser(SysUser sysUser) {
        userMapper.insertSelective(sysUser);
        return sysUser;
    }

    @Override
    @CachePut(value = "user", key="'user_'+#sysUser.id")
    public SysUser updateUser(SysUser sysUser) {
        userMapper.updateByPrimaryKeySelective(sysUser);
        return sysUser;
    }

    @Override
    @CacheEvict(value = "user", key="'user_'+#userId")
    public void deleteUser(Long userId) {
        userMapper.deleteByPrimaryKey(userId);
    }
}
```

