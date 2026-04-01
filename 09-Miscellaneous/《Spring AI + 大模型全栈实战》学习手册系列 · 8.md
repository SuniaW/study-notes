# 专题八：《RAG 系统安全与权限管理：企业级数据保护方案》

## 📚 企业级 RAG 智能问答系统全栈实施指南

### 第八部分：安全与权限管理

---

## 第 1 章 企业级安全架构设计

### 1.1 RAG 系统安全威胁模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RAG 系统安全威胁矩阵                                │
├─────────────────────────────────────────────────────────────────────────┤
│  威胁类型      │ 攻击场景                  │ 风险等级 │ 防护措施        │
├─────────────────────────────────────────────────────────────────────────┤
│  未授权访问    │ 越权查询敏感文档          │ 🔴 P0    │ RBAC + ABAC    │
│  数据泄露      │ 向量数据库被导出          │ 🔴 P0    │ 加密存储        │
│  提示词注入    │ 恶意 Prompt 绕过权限       │ 🔴 P0    │ 输入过滤        │
│  模型投毒      │ 污染训练/检索数据         │ 🟠 P1    │ 数据校验        │
│  会话劫持      │ Token 盗用                │ 🟠 P1    │ JWT + 刷新机制  │
│  审计缺失      │ 操作无法追溯              │ 🟡 P2    │ 全链路日志      │
│  DDoS 攻击     │ 大量请求耗尽资源          │ 🟡 P2    │ 限流 + 熔断     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 安全架构分层设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        企业级 RAG 安全架构                               │
├─────────────────────────────────────────────────────────────────────────┤
│  L1: 接入层 → WAF + 限流 + IP 黑白名单 + SSL/TLS                         │
│     ↓                                                                   │
│  L2: 认证层 → SSO 单点登录 + JWT/OAuth2 + MFA 多因素认证                  │
│     ↓                                                                   │
│  L3: 授权层 → RBAC 角色权限 + ABAC 属性权限 + 文档级权限                  │
│     ↓                                                                   │
│  L4: 数据层 → 字段加密 + 数据脱敏 + 向量加密 + 审计日志                  │
│     ↓                                                                   │
│  L5: 模型层 → Prompt 过滤 + 输出审查 + 敏感词检测                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 安全合规要求对照表

| 合规标准 | 适用场景 | 核心要求 | RAG 系统对应措施 |
|----------|----------|----------|------------------|
| **等保 2.0** | 中国政府/国企 | 三级等保 | 身份认证、访问控制、审计日志 |
| **GDPR** | 欧盟用户数据 | 数据保护 | 数据脱敏、删除权、可携带权 |
| **ISO 27001** | 国际企业 | 信息安全 | 安全策略、风险评估、持续改进 |
| **SOC 2** | SaaS 服务 | 信任服务 | 可用性、保密性、隐私保护 |
| **个人信息保护法** | 中国用户数据 | 隐私保护 | 最小化收集、知情同意、安全存储 |

---

## 第 2 章 RBAC 权限模型设计与实现

### 2.1 RBAC 权限模型架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         RBAC 权限模型                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  用户 (User) → 角色 (Role) → 权限 (Permission) → 资源 (Resource)         │
│     ↓           ↓              ↓                ↓                        │
│   张三      部门经理      文档读取        财务部文档                      │
│   李四      普通员工      文档查询        公开文档                        │
│   王五      系统管理员    全部权限        全部资源                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据库表设计

```sql
-- 用户表
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    department_id BIGINT,
    status TINYINT DEFAULT 1,  -- 1:正常 0:禁用
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_department (department_id),
    INDEX idx_status (status)
);

-- 角色表
CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    role_code VARCHAR(50) UNIQUE NOT NULL,
    role_name VARCHAR(100) NOT NULL,
    description VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_code (role_code)
);

-- 权限表
CREATE TABLE sys_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    perm_code VARCHAR(100) UNIQUE NOT NULL,
    perm_name VARCHAR(100) NOT NULL,
    resource_type VARCHAR(20),  -- MENU/API/DOCUMENT
    resource_path VARCHAR(255),
    parent_id BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_code (perm_code)
);

-- 用户角色关联表
CREATE TABLE sys_user_role (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES sys_user(id),
    FOREIGN KEY (role_id) REFERENCES sys_role(id)
);

-- 角色权限关联表
CREATE TABLE sys_role_permission (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES sys_role(id),
    FOREIGN KEY (permission_id) REFERENCES sys_permission(id)
);

-- 文档权限表（文档级权限控制）
CREATE TABLE doc_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    doc_id VARCHAR(100) NOT NULL,
    department_id BIGINT,
    role_id BIGINT,
    user_id BIGINT,
    permission_level TINYINT NOT NULL,  -- 1:可读 2:可写 3:完全控制
    expire_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_doc_user (doc_id, user_id),
    INDEX idx_doc (doc_id),
    INDEX idx_department (department_id)
);
```

### 2.3 Spring Security + JWT 实现

```java
// SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                // 公开接口
                .requestMatchers("/api/auth/**", "/api/public/**").permitAll()
                // 需要认证的接口
                .requestMatchers("/api/rag/**").authenticated()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                // 其他接口需要认证
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authenticationProvider(authenticationProvider)
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
            )
            .build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://rag.yourdomain.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}

// JwtAuthenticationFilter.java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        try {
            // 1. 从请求头获取 Token
            String token = extractToken(request);
            
            if (token != null && jwtTokenProvider.validateToken(token)) {
                // 2. 解析 Token 获取用户名
                String username = jwtTokenProvider.getUsername(token);
                
                // 3. 加载用户详情
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                // 4. 设置认证上下文
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );
                
                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );
                
                SecurityContextHolder.getContext().setAuthentication(authentication);
                
                // 5. 记录审计日志
                AuditLogContext.setUserId(userDetails.getUsername());
                AuditLogContext.setRequestId(UUID.randomUUID().toString());
            }
        } catch (Exception e) {
            log.error("认证失败", e);
        }
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            // 清理 ThreadLocal
            AuditLogContext.clear();
        }
    }
    
    private String extractToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (bearer != null && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
}

// JwtTokenProvider.java
@Component
@RequiredArgsConstructor
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration:3600000}")
    private long jwtExpiration;
    
    @Value("${jwt.refresh-expiration:604800000}")
    private long refreshExpiration;
    
    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
    
    public String generateToken(UserDetails userDetails) {
        AppUser appUser = (AppUser) userDetails;
        
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .claim("userId", appUser.getId())
            .claim("roles", appUser.getRoles().stream()
                .map(Role::getRoleCode)
                .collect(Collectors.toList()))
            .claim("departmentId", appUser.getDepartmentId())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + refreshExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }
    
    public String getUsername(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    public boolean isTokenExpired(String token) {
        Date expiration = Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getExpiration();
        return expiration.before(new Date());
    }
}
```

### 2.4 权限注解与 AOP 实现

```java
// @RequirePermission 自定义注解
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequirePermission {
    String value();  // 权限码
    Logical logical() default Logical.AND;  // AND/OR
}

// PermissionAspect.java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class PermissionAspect {
    
    private final PermissionService permissionService;
    
    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint pjp, 
                                   RequirePermission requirePermission) throws Throwable {
        // 1. 获取当前用户
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication == null || !authentication.isAuthenticated()) {
            throw new AccessDeniedException("未认证");
        }
        
        String username = authentication.getName();
        String permissionCode = requirePermission.value();
        
        // 2. 检查权限
        boolean hasPermission = permissionService.hasPermission(username, permissionCode);
        
        if (!hasPermission) {
            log.warn("用户 {} 无权访问权限 {}", username, permissionCode);
            
            // 3. 记录审计日志
            AuditLog auditLog = AuditLog.builder()
                .userId(username)
                .action("PERMISSION_DENIED")
                .resource(permissionCode)
                .status("FAILED")
                .reason("权限不足")
                .timestamp(LocalDateTime.now())
                .build();
            auditLogService.save(auditLog);
            
            throw new AccessDeniedException("权限不足");
        }
        
        // 4. 执行目标方法
        return pjp.proceed();
    }
}

// 使用示例
@RestController
@RequestMapping("/api/rag")
@RequiredArgsConstructor
public class RagController {
    
    @PostMapping("/query")
    @RequirePermission("rag:query")
    public ResponseEntity<RagResponse> query(@RequestBody ChatRequest request) {
        RagResponse response = ragService.query(request.getQuestion());
        return ResponseEntity.ok(response);
    }
    
    @PostMapping("/document/upload")
    @RequirePermission("rag:document:upload")
    public ResponseEntity<Void> uploadDocument(@RequestParam("file") MultipartFile file) {
        documentService.upload(file);
        return ResponseEntity.ok().build();
    }
    
    @DeleteMapping("/document/{docId}")
    @RequirePermission("rag:document:delete")
    public ResponseEntity<Void> deleteDocument(@PathVariable String docId) {
        documentService.delete(docId);
        return ResponseEntity.ok().build();
    }
}
```

---

## 第 3 章 文档级权限控制

### 3.1 文档权限模型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       文档级权限控制模型                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  文档 → 部门权限 + 角色权限 + 用户权限 + 时间有效期                       │
│                                                                        │
│  示例：                                                                 │
│  文档 A → 财务部 (可读) + 部门经理 (可写) + 张三 (完全控制) + 30 天有效    │
│  文档 B → 全公司 (可读) + 管理员 (可写) + 永久有效                       │
│  文档 C → 仅本人 (可读可写) + 7 天有效                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 文档权限服务实现

```java
// DocumentPermissionService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class DocumentPermissionService {
    
    private final DocumentPermissionRepository permissionRepository;
    private final UserService userService;
    
    /**
     * 检查用户对文档的访问权限
     */
    public boolean hasAccess(String userId, String docId, PermissionLevel requiredLevel) {
        // 1. 检查是否是文档所有者
        if (isDocumentOwner(userId, docId)) {
            return true;
        }
        
        // 2. 检查是否是系统管理员
        if (isSystemAdmin(userId)) {
            return true;
        }
        
        // 3. 检查用户级权限
        Optional<DocumentPermission> userPerm = permissionRepository
            .findByDocIdAndUserId(docId, userId);
        if (userPerm.isPresent() && !userPerm.get().isExpired() 
            && userPerm.get().getPermissionLevel() >= requiredLevel.getValue()) {
            return true;
        }
        
        // 4. 检查角色级权限
        User user = userService.findById(userId);
        for (Role role : user.getRoles()) {
            Optional<DocumentPermission> rolePerm = permissionRepository
                .findByDocIdAndRoleId(docId, role.getId());
            if (rolePerm.isPresent() && !rolePerm.get().isExpired()
                && rolePerm.get().getPermissionLevel() >= requiredLevel.getValue()) {
                return true;
            }
        }
        
        // 5. 检查部门级权限
        Optional<DocumentPermission> deptPerm = permissionRepository
            .findByDocIdAndDepartmentId(docId, user.getDepartmentId());
        if (deptPerm.isPresent() && !deptPerm.get().isExpired()
            && deptPerm.get().getPermissionLevel() >= requiredLevel.getValue()) {
            return true;
        }
        
        return false;
    }
    
    /**
     * 获取用户可访问的文档 ID 列表（用于向量检索过滤）
     */
    public List<String> getAccessibleDocIds(String userId) {
        User user = userService.findById(userId);
        Set<String> docIds = new HashSet<>();
        
        // 1. 用户级权限文档
        List<DocumentPermission> userPerms = permissionRepository.findByUserId(userId);
        userPerms.stream()
            .filter(p -> !p.isExpired())
            .forEach(p -> docIds.add(p.getDocId()));
        
        // 2. 角色级权限文档
        for (Role role : user.getRoles()) {
            List<DocumentPermission> rolePerms = permissionRepository.findByRoleId(role.getId());
            rolePerms.stream()
                .filter(p -> !p.isExpired())
                .forEach(p -> docIds.add(p.getDocId()));
        }
        
        // 3. 部门级权限文档
        List<DocumentPermission> deptPerms = permissionRepository.findByDepartmentId(user.getDepartmentId());
        deptPerms.stream()
            .filter(p -> !p.isExpired())
            .forEach(p -> docIds.add(p.getDocId()));
        
        // 4. 公开文档
        List<DocumentPermission> publicPerms = permissionRepository.findByUserId(null);
        publicPerms.stream()
            .filter(p -> !p.isExpired())
            .forEach(p -> docIds.add(p.getDocId()));
        
        return new ArrayList<>(docIds);
    }
    
    /**
     * 设置文档权限
     */
    public void setPermission(String docId, PermissionRequest request) {
        DocumentPermission permission = DocumentPermission.builder()
            .docId(docId)
            .userId(request.getUserId())
            .roleId(request.getRoleId())
            .departmentId(request.getDepartmentId())
            .permissionLevel(request.getPermissionLevel())
            .expireAt(request.getExpireAt())
            .build();
        
        permissionRepository.save(permission);
        log.info("文档权限已设置：docId={}, target={}, level={}", 
                docId, request.getTarget(), request.getPermissionLevel());
    }
}

// PermissionLevel 枚举
public enum PermissionLevel {
    READ(1, "可读"),
    WRITE(2, "可写"),
    FULL_CONTROL(3, "完全控制");
    
    private final int value;
    private final String description;
    
    PermissionLevel(int value, String description) {
        this.value = value;
        this.description = description;
    }
    
    public int getValue() { return value; }
}

// PermissionRequest DTO
@Data
@Builder
public class PermissionRequest {
    private String userId;
    private Long roleId;
    private Long departmentId;
    private PermissionLevel permissionLevel;
    private LocalDateTime expireAt;
    
    public String getTarget() {
        if (userId != null) return "USER:" + userId;
        if (roleId != null) return "ROLE:" + roleId;
        if (departmentId != null) return "DEPT:" + departmentId;
        return "PUBLIC";
    }
}
```

### 3.3 向量检索权限过滤

```java
// SecureVectorStore.java
@Service
@RequiredArgsConstructor
@Slf4j
public class SecureVectorStore {
    
    private final VectorStore vectorStore;
    private final DocumentPermissionService permissionService;
    
    /**
     * 带权限过滤的向量检索
     */
    public List<Document> similaritySearch(String userId, 
                                           String query, 
                                           int topK) {
        // 1. 获取用户可访问的文档 ID 列表
        List<String> accessibleDocIds = permissionService.getAccessibleDocIds(userId);
        
        if (accessibleDocIds.isEmpty()) {
            log.warn("用户 {} 无任何文档访问权限", userId);
            return Collections.emptyList();
        }
        
        // 2. 构建过滤条件
        SearchRequest request = SearchRequest.query(query)
            .withTopK(topK * 2)  // 多检索一些，过滤后保证 topK
            .withFilterExpression(buildFilterExpression(accessibleDocIds));
        
        // 3. 执行检索
        List<Document> results = vectorStore.similaritySearch(request);
        
        // 4. 二次权限校验（防御性编程）
        List<Document> authorizedResults = results.stream()
            .filter(doc -> permissionService.hasAccess(userId, doc.getId(), PermissionLevel.READ))
            .limit(topK)
            .collect(Collectors.toList());
        
        log.info("向量检索：用户={}, 原始结果={}, 过滤后={}", 
                userId, results.size(), authorizedResults.size());
        
        return authorizedResults;
    }
    
    /**
     * 构建 Milvus 过滤表达式
     */
    private String buildFilterExpression(List<String> docIds) {
        // Milvus 过滤表达式语法：doc_id in ["id1", "id2", ...]
        String ids = docIds.stream()
            .map(id -> "\"" + id + "\"")
            .collect(Collectors.joining(", "));
        return "doc_id in [" + ids + "]";
    }
}
```

### 3.4 文档权限管理界面

```vue
<!-- DocumentPermissionModal.vue -->
<template>
  <el-dialog v-model="visible" title="文档权限管理" width="600px">
    <el-form :model="form" label-width="100px">
      <el-form-item label="权限类型">
        <el-radio-group v-model="form.targetType">
          <el-radio label="user">用户</el-radio>
          <el-radio label="role">角色</el-radio>
          <el-radio label="department">部门</el-radio>
          <el-radio label="public">公开</el-radio>
        </el-radio-group>
      </el-form-item>
      
      <el-form-item v-if="form.targetType === 'user'" label="选择用户">
        <el-select v-model="form.userId" placeholder="请选择用户" filterable>
          <el-option
            v-for="user in users"
            :key="user.id"
            :label="user.username"
            :value="user.id"
          />
        </el-select>
      </el-form-item>
      
      <el-form-item v-if="form.targetType === 'role'" label="选择角色">
        <el-select v-model="form.roleId" placeholder="请选择角色">
          <el-option
            v-for="role in roles"
            :key="role.id"
            :label="role.roleName"
            :value="role.id"
          />
        </el-select>
      </el-form-item>
      
      <el-form-item v-if="form.targetType === 'department'" label="选择部门">
        <el-select v-model="form.departmentId" placeholder="请选择部门">
          <el-option
            v-for="dept in departments"
            :key="dept.id"
            :label="dept.name"
            :value="dept.id"
          />
        </el-select>
      </el-form-item>
      
      <el-form-item label="权限级别">
        <el-select v-model="form.permissionLevel">
          <el-option label="可读" :value="1" />
          <el-option label="可写" :value="2" />
          <el-option label="完全控制" :value="3" />
        </el-select>
      </el-form-item>
      
      <el-form-item label="有效期">
        <el-date-picker
          v-model="form.expireAt"
          type="datetime"
          placeholder="选择过期时间"
          :clearable="true"
        />
      </el-form-item>
    </el-form>
    
    <template #footer>
      <el-button @click="visible = false">取消</el-button>
      <el-button type="primary" @click="handleSubmit">确定</el-button>
    </template>
  </el-dialog>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import { ElMessage } from 'element-plus'

const visible = defineModel<boolean>('visible', { default: false })
const emit = defineEmits<{ success: [] }>()

const form = reactive({
  targetType: 'user',
  userId: null,
  roleId: null,
  departmentId: null,
  permissionLevel: 1,
  expireAt: null
})

const users = ref([])
const roles = ref([])
const departments = ref([])

const handleSubmit = async () => {
  try {
    await permissionApi.setPermission(props.docId, form)
    ElMessage.success('权限设置成功')
    visible.value = false
    emit('success')
  } catch (error) {
    ElMessage.error('设置失败')
  }
}
</script>
```

---

## 第 4 章 数据脱敏与加密

### 4.1 敏感数据识别规则

```java
// SensitiveDataPattern.java
@Component
public class SensitiveDataPattern {
    
    // 身份证号
    public static final Pattern ID_CARD = Pattern.compile(
        "(^[1-9]\\d{5}(18|19|20)\\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\\d|3[01])\\d{3}[\\dXx]$)"
    );
    
    // 手机号
    public static final Pattern PHONE = Pattern.compile(
        "(^1[3-9]\\d{9}$)"
    );
    
    // 邮箱
    public static final Pattern EMAIL = Pattern.compile(
        "([a-zA-Z0-9._-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+)"
    );
    
    // 银行卡号
    public static final Pattern BANK_CARD = Pattern.compile(
        "(^\\d{16,19}$)"
    );
    
    // 地址（简化）
    public static final Pattern ADDRESS = Pattern.compile(
        "(.*?[省市区县].*?[路街道].*?)"
    );
}

// SensitiveDataTypes 枚举
public enum SensitiveDataType {
    ID_CARD("身份证号", 5, 5),      // 前 5 后 5
    PHONE("手机号", 3, 4),          // 前 3 后 4
    EMAIL("邮箱", 2, 4),            // 前 2 后 4
    BANK_CARD("银行卡", 4, 4),      // 前 4 后 4
    ADDRESS("地址", 2, 2),          // 前 2 后 2
    NAME("姓名", 1, 1);             // 姓 +*+ 名
    
    private final String description;
    private final int keepPrefix;
    private final int keepSuffix;
    
    SensitiveDataType(String description, int keepPrefix, int keepSuffix) {
        this.description = description;
        this.keepPrefix = keepPrefix;
        this.keepSuffix = keepSuffix;
    }
}
```

### 4.2 数据脱敏服务

```java
// DataMaskingService.java
@Service
@Slf4j
public class DataMaskingService {
    
    /**
     * 智能脱敏（自动识别敏感数据类型）
     */
    public String mask(String text) {
        if (text == null || text.isEmpty()) {
            return text;
        }
        
        String result = text;
        
        // 身份证脱敏
        result = maskPattern(result, SensitiveDataPattern.ID_CARD, SensitiveDataType.ID_CARD);
        
        // 手机号脱敏
        result = maskPattern(result, SensitiveDataPattern.PHONE, SensitiveDataType.PHONE);
        
        // 邮箱脱敏
        result = maskPattern(result, SensitiveDataPattern.EMAIL, SensitiveDataType.EMAIL);
        
        // 银行卡脱敏
        result = maskPattern(result, SensitiveDataPattern.BANK_CARD, SensitiveDataType.BANK_CARD);
        
        return result;
    }
    
    /**
     * 按模式脱敏
     */
    private String maskPattern(String text, Pattern pattern, SensitiveDataType type) {
        Matcher matcher = pattern.matcher(text);
        StringBuffer sb = new StringBuffer();
        
        while (matcher.find()) {
            String matched = matcher.group();
            String masked = maskValue(matched, type);
            matcher.appendReplacement(sb, masked);
        }
        matcher.appendTail(sb);
        
        return sb.toString();
    }
    
    /**
     * 脱敏具体值
     */
    private String maskValue(String value, SensitiveDataType type) {
        if (value.length() <= type.keepPrefix + type.keepSuffix) {
            return "*".repeat(value.length());
        }
        
        String prefix = value.substring(0, type.keepPrefix);
        String suffix = value.substring(value.length() - type.keepSuffix);
        int maskLength = value.length() - type.keepPrefix - type.keepSuffix;
        
        return prefix + "*".repeat(maskLength) + suffix;
    }
    
    /**
     * 文档内容脱敏（用于向量存储前）
     */
    public String maskDocumentContent(String content) {
        // 可选：是否对向量库存储的内容进行脱敏
        // 方案 1：完全脱敏（安全性高，检索精度略降）
        // 方案 2：保留原始内容，检索结果脱敏（推荐）
        return content;  // 方案 2：存储原始内容
    }
    
    /**
     * 检索结果脱敏（返回给用户前）
     */
    public Document maskDocument(Document doc) {
        String maskedContent = mask(doc.getContent());
        return new Document(
            doc.getId(),
            maskedContent,
            doc.getMetadata()
        );
    }
}

// 使用示例
@RestController
@RequestMapping("/api/rag")
@RequiredArgsConstructor
public class RagController {
    
    private final SecureVectorStore vectorStore;
    private final DataMaskingService maskingService;
    
    @PostMapping("/query")
    public ResponseEntity<RagResponse> query(@RequestBody ChatRequest request) {
        String userId = SecurityContextHolder.getContext().getAuthentication().getName();
        
        // 1. 带权限的向量检索
        List<Document> docs = vectorStore.similaritySearch(userId, request.getQuestion(), 5);
        
        // 2. 检索结果脱敏
        List<Document> maskedDocs = docs.stream()
            .map(maskingService::maskDocument)
            .collect(Collectors.toList());
        
        // 3. 生成回答
        String answer = ragService.generateAnswer(request.getQuestion(), maskedDocs);
        
        RagResponse response = new RagResponse(answer, maskedDocs, false);
        return ResponseEntity.ok(response);
    }
}
```

### 4.3 字段级加密（AES-256）

```java
// FieldEncryptionAspect.java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class FieldEncryptionAspect {
    
    private final EncryptionService encryptionService;
    
    /**
     * 加密标注字段
     */
    @Around("@annotation(EncryptField)")
    public Object encryptFields(ProceedingJoinPoint pjp, EncryptField annotation) throws Throwable {
        Object result = pjp.proceed();
        
        if (result == null) {
            return null;
        }
        
        // 递归处理返回对象
        encryptObject(result, annotation.fields());
        
        return result;
    }
    
    /**
     * 解密标注字段
     */
    @Around("@annotation(DecryptField)")
    public Object decryptFields(ProceedingJoinPoint pjp, DecryptField annotation) throws Throwable {
        Object result = pjp.proceed();
        
        if (result == null) {
            return null;
        }
        
        decryptObject(result, annotation.fields());
        
        return result;
    }
    
    private void encryptObject(Object obj, String[] fields) {
        if (obj == null) return;
        
        Class<?> clazz = obj.getClass();
        for (String fieldName : fields) {
            try {
                Field field = clazz.getDeclaredField(fieldName);
                field.setAccessible(true);
                Object value = field.get(obj);
                if (value instanceof String) {
                    String encrypted = encryptionService.encrypt((String) value);
                    field.set(obj, encrypted);
                }
            } catch (Exception e) {
                log.warn("加密字段失败：{}", fieldName, e);
            }
        }
    }
}

// EncryptionService.java
@Service
@Slf4j
public class EncryptionService {
    
    @Value("${encryption.aes-key}")
    private String aesKey;
    
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_TAG_LENGTH = 128;
    private static final int GCM_IV_LENGTH = 12;
    
    /**
     * AES-256-GCM 加密
     */
    public String encrypt(String plaintext) throws Exception {
        if (plaintext == null) return null;
        
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        byte[] iv = generateSecureRandomBytes(GCM_IV_LENGTH);
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        
        cipher.init(Cipher.ENCRYPT_MODE, getSecretKey(), spec);
        byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
        
        // IV + Ciphertext 一起存储
        byte[] combined = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, combined, 0, iv.length);
        System.arraycopy(ciphertext, 0, combined, iv.length, ciphertext.length);
        
        return Base64.getEncoder().encodeToString(combined);
    }
    
    /**
     * AES-256-GCM 解密
     */
    public String decrypt(String ciphertext) throws Exception {
        if (ciphertext == null) return null;
        
        byte[] combined = Base64.getDecoder().decode(ciphertext);
        byte[] iv = Arrays.copyOfRange(combined, 0, GCM_IV_LENGTH);
        byte[] actualCiphertext = Arrays.copyOfRange(combined, GCM_IV_LENGTH, combined.length);
        
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        
        cipher.init(Cipher.DECRYPT_MODE, getSecretKey(), spec);
        byte[] plaintext = cipher.doFinal(actualCiphertext);
        
        return new String(plaintext, StandardCharsets.UTF_8);
    }
    
    private SecretKey getSecretKey() {
        byte[] keyBytes = Decoders.BASE64.decode(aesKey);
        return new SecretKeySpec(keyBytes, "AES");
    }
    
    private byte[] generateSecureRandomBytes(int length) {
        SecureRandom secureRandom = new SecureRandom();
        byte[] bytes = new byte[length];
        secureRandom.nextBytes(bytes);
        return bytes;
    }
}

// 使用示例
@Data
public class DocumentDTO {
    private String id;
    
    @EncryptField  // 自定义注解
    private String content;
    
    private Map<String, Object> metadata;
}
```

---

## 第 5 章 审计日志系统

### 5.1 审计日志数据模型

```sql
-- 审计日志表
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    request_id VARCHAR(100) NOT NULL,      -- 请求 ID（链路追踪）
    user_id VARCHAR(100) NOT NULL,          -- 用户 ID
    username VARCHAR(100),                  -- 用户名
    action VARCHAR(100) NOT NULL,           -- 操作类型
    resource VARCHAR(255),                  -- 操作资源
    resource_id VARCHAR(100),               -- 资源 ID
    method VARCHAR(20),                     -- HTTP 方法
    uri VARCHAR(500),                       -- 请求 URI
    ip_address VARCHAR(50),                 -- 客户端 IP
    user_agent VARCHAR(500),                -- 用户代理
    request_body TEXT,                      -- 请求体（脱敏）
    response_status INT,                    -- 响应状态码
    response_body TEXT,                     -- 响应体（脱敏）
    latency_ms INT,                         -- 请求耗时（毫秒）
    status VARCHAR(20),                     -- SUCCESS/FAILED
    reason VARCHAR(500),                    -- 失败原因
    risk_level TINYINT DEFAULT 0,           -- 风险等级 0-5
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id),
    INDEX idx_action (action),
    INDEX idx_resource (resource),
    INDEX idx_created (created_at),
    INDEX idx_status (status),
    INDEX idx_risk (risk_level)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 审计日志分区（按月分区）
ALTER TABLE audit_log 
PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202601 VALUES LESS THAN (202602),
    PARTITION p202602 VALUES LESS THAN (202603),
    PARTITION p202603 VALUES LESS THAN (202604),
    -- ... 更多分区
    PARTITION pmax VALUES LESS THAN MAXVALUE
);
```

### 5.2 审计日志切面

```java
// AuditLogAspect.java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class AuditLogAspect {
    
    private final AuditLogService auditLogService;
    private final DataMaskingService maskingService;
    
    @Around("@annotation(auditLog)")
    public Object logAudit(ProceedingJoinPoint pjp, AuditLog auditLog) throws Throwable {
        long startTime = System.currentTimeMillis();
        String requestId = AuditLogContext.getRequestId();
        String userId = AuditLogContext.getUserId();
        
        HttpServletRequest request = getRequest();
        
        // 构建审计日志
        AuditLogEntry entry = AuditLogEntry.builder()
            .requestId(requestId)
            .userId(userId)
            .action(auditLog.action())
            .resource(auditLog.resource())
            .method(request != null ? request.getMethod() : null)
            .uri(request != null ? request.getRequestURI() : null)
            .ipAddress(getClientIp(request))
            .userAgent(request != null ? request.getHeader("User-Agent") : null)
            .startTime(startTime)
            .build();
        
        try {
            // 记录请求体（脱敏）
            Object[] args = pjp.getArgs();
            if (args.length > 0) {
                String requestBody = maskSensitiveData(JsonMapper.toJson(args[0]));
                entry.setRequestBody(requestBody);
            }
            
            // 执行目标方法
            Object result = pjp.proceed();
            
            // 记录响应
            long endTime = System.currentTimeMillis();
            entry.setLatencyMs((int) (endTime - startTime));
            entry.setResponseStatus(200);
            entry.setStatus("SUCCESS");
            entry.setResponseBody(maskSensitiveData(JsonMapper.toJson(result)));
            
            // 异步保存日志
            auditLogService.saveAsync(entry);
            
            return result;
            
        } catch (Exception e) {
            long endTime = System.currentTimeMillis();
            entry.setLatencyMs((int) (endTime - startTime));
            entry.setResponseStatus(500);
            entry.setStatus("FAILED");
            entry.setReason(e.getMessage());
            
            // 异步保存日志
            auditLogService.saveAsync(entry);
            
            throw e;
        }
    }
    
    private String maskSensitiveData(String json) {
        if (json == null || json.length() > 10000) {
            return null;  // 过大不记录
        }
        return maskingService.mask(json);
    }
    
    private HttpServletRequest getRequest() {
        try {
            return ((ServletRequestAttributes) RequestContextHolder
                .currentRequestAttributes()).getRequest();
        } catch (Exception e) {
            return null;
        }
    }
    
    private String getClientIp(HttpServletRequest request) {
        if (request == null) return null;
        
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("X-Real-IP");
        }
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}

// @AuditLog 自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AuditLog {
    String action();      // 操作类型
    String resource();    // 操作资源
}

// 使用示例
@RestController
@RequestMapping("/api/rag")
@RequiredArgsConstructor
public class RagController {
    
    @PostMapping("/query")
    @AuditLog(action = "RAG_QUERY", resource = "rag:document")
    public ResponseEntity<RagResponse> query(@RequestBody ChatRequest request) {
        // ...
    }
    
    @PostMapping("/document/upload")
    @AuditLog(action = "DOCUMENT_UPLOAD", resource = "rag:document")
    public ResponseEntity<Void> upload(@RequestParam("file") MultipartFile file) {
        // ...
    }
    
    @DeleteMapping("/document/{docId}")
    @AuditLog(action = "DOCUMENT_DELETE", resource = "rag:document")
    public ResponseEntity<Void> delete(@PathVariable String docId) {
        // ...
    }
}
```

### 5.3 审计日志查询 API

```java
// AuditLogController.java
@RestController
@RequestMapping("/api/admin/audit")
@RequiredArgsConstructor
@RequirePermission("admin:audit:view")
public class AuditLogController {
    
    private final AuditLogService auditLogService;
    
    /**
     * 查询审计日志
     */
    @GetMapping("/logs")
    public ResponseEntity<PageResult<AuditLogVO>> queryLogs(
            @RequestParam(required = false) String userId,
            @RequestParam(required = false) String action,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) Integer riskLevel,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startTime,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endTime,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        AuditLogQuery query = AuditLogQuery.builder()
            .userId(userId)
            .action(action)
            .status(status)
            .riskLevel(riskLevel)
            .startTime(startTime)
            .endTime(endTime)
            .page(page)
            .size(size)
            .build();
        
        PageResult<AuditLogVO> result = auditLogService.query(query);
        return ResponseEntity.ok(result);
    }
    
    /**
     * 导出审计日志
     */
    @PostMapping("/export")
    public ResponseEntity<Void> exportLogs(
            @RequestBody AuditLogQuery query,
            HttpServletResponse response) throws IOException {
        
        response.setContentType("application/vnd.ms-excel");
        response.setHeader("Content-Disposition", 
            "attachment; filename=audit_logs_" + System.currentTimeMillis() + ".xlsx");
        
        auditLogService.export(query, response.getOutputStream());
        return ResponseEntity.ok().build();
    }
    
    /**
     * 获取审计统计
     */
    @GetMapping("/statistics")
    public ResponseEntity<AuditStatistics> getStatistics(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime startTime,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) LocalDateTime endTime) {
        
        AuditStatistics stats = auditLogService.getStatistics(startTime, endTime);
        return ResponseEntity.ok(stats);
    }
}
```

### 5.4 高风险操作告警

```java
// RiskAlertService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class RiskAlertService {
    
    private final AuditLogService auditLogService;
    private final AlertNotificationService notificationService;
    
    // 高风险操作定义
    private static final Set<String> HIGH_RISK_ACTIONS = Set.of(
        "DOCUMENT_DELETE",
        "DOCUMENT_EXPORT",
        "PERMISSION_CHANGE",
        "USER_ROLE_CHANGE",
        "BATCH_QUERY"
    );
    
    /**
     * 检查并触发风险告警
     */
    @Async
    @EventListener
    public void handleAuditLog(AuditLogEvent event) {
        AuditLogEntry entry = event.getEntry();
        
        // 1. 计算风险等级
        int riskLevel = calculateRiskLevel(entry);
        entry.setRiskLevel(riskLevel);
        
        // 2. 高风险操作告警
        if (riskLevel >= 4) {
            sendHighRiskAlert(entry);
        }
        
        // 3. 异常行为检测
        detectAnomaly(entry);
    }
    
    /**
     * 计算风险等级（0-5）
     */
    private int calculateRiskLevel(AuditLogEntry entry) {
        int risk = 0;
        
        // 高风险操作 +2
        if (HIGH_RISK_ACTIONS.contains(entry.getAction())) {
            risk += 2;
        }
        
        // 失败操作 +1
        if ("FAILED".equals(entry.getStatus())) {
            risk += 1;
        }
        
        // 非工作时间 +1
        if (isNonWorkingHour(entry.getCreatedAt())) {
            risk += 1;
        }
        
        // 非常用 IP +1
        if (isUnusualIp(entry.getUserId(), entry.getIpAddress())) {
            risk += 1;
        }
        
        // 频繁操作 +1
        if (isFrequentOperation(entry.getUserId(), entry.getAction())) {
            risk += 1;
        }
        
        return Math.min(risk, 5);
    }
    
    /**
     * 发送高风险告警
     */
    private void sendHighRiskAlert(AuditLogEntry entry) {
        Alert alert = Alert.builder()
            .alertName("HIGH_RISK_OPERATION")
            .severity("CRITICAL")
            .instance(entry.getUserId())
            .currentValue(entry.getAction())
            .threshold("Risk Level >= 4")
            .duration("Immediate")
            .timestamp(entry.getCreatedAt())
            .build();
        
        // 钉钉告警
        notificationService.sendDingTalkAlert(alert);
        
        // 邮件告警（安全团队）
        notificationService.sendEmailAlert(alert, securityTeamEmails);
        
        log.warn("高风险操作告警：userId={}, action={}, riskLevel={}", 
                entry.getUserId(), entry.getAction(), entry.getRiskLevel());
    }
    
    /**
     * 异常行为检测
     */
    private void detectAnomaly(AuditLogEntry entry) {
        // 1. 检测短时间内大量查询
        long queryCount = auditLogService.countByUserAndAction(
            entry.getUserId(), "RAG_QUERY", 
            entry.getCreatedAt().minusMinutes(5), entry.getCreatedAt()
        );
        
        if (queryCount > 100) {
            log.warn("异常行为检测：用户 {} 5 分钟内查询 {} 次", entry.getUserId(), queryCount);
            // 触发限流或临时封禁
        }
        
        // 2. 检测敏感文档访问
        // ...
    }
}
```

---

## 第 6 章 安全测试与合规验证

### 6.1 安全测试清单

```markdown
## 🔒 RAG 系统安全测试清单

### 认证与授权
- [ ] 未登录用户无法访问受保护接口
- [ ] Token 过期后自动拒绝访问
- [ ] 无效 Token 返回 401
- [ ] 越权访问返回 403
- [ ] 管理员权限正确隔离

### 文档权限
- [ ] 用户只能访问授权文档
- [ ] 部门权限正确生效
- [ ] 角色权限正确生效
- [ ] 权限过期后自动失效
- [ ] 向量检索结果正确过滤

### 数据脱敏
- [ ] 身份证号正确脱敏
- [ ] 手机号正确脱敏
- [ ] 邮箱正确脱敏
- [ ] 检索结果脱敏生效
- [ ] 日志中敏感信息脱敏

### 审计日志
- [ ] 所有操作都有日志记录
- [ ] 日志无法被篡改
- [ ] 日志保留期限符合要求
- [ ] 高风险操作触发告警
- [ ] 日志查询功能正常

### 渗透测试
- [ ] SQL 注入测试通过
- [ ] XSS 攻击测试通过
- [ ] CSRF 攻击测试通过
- [ ] 提示词注入测试通过
- [ ] 暴力破解防护生效
```

### 6.2 提示词注入防护

```java
// PromptInjectionFilter.java
@Component
@Slf4j
public class PromptInjectionFilter {
    
    // 常见提示词注入模式
    private static final List<Pattern> INJECTION_PATTERNS = List.of(
        Pattern.compile("(?i)ignore\\s+previous\\s+instructions"),
        Pattern.compile("(?i)you\\s+are\\s+now\\s+in\\s+developer\\s+mode"),
        Pattern.compile("(?i)bypass\\s+(all|any)\\s+(rules|restrictions)"),
        Pattern.compile("(?i)output\\s+(your|the)\\s+(system|initial)\\s+prompt"),
        Pattern.compile("(?i)print\\s+(your|the)\\s+(instructions|rules)"),
        Pattern.compile("(?i)act\\s+as\\s+(another|different)\\s+(ai|model)"),
        Pattern.compile("(?i)disable\\s+(safety|security)\\s+(filters|checks)")
    );
    
    /**
     * 检测并过滤提示词注入
     */
    public boolean isSafe(String input) {
        if (input == null || input.isEmpty()) {
            return true;
        }
        
        for (Pattern pattern : INJECTION_PATTERNS) {
            if (pattern.matcher(input).find()) {
                log.warn("检测到提示词注入尝试：{}", input.substring(0, Math.min(100, input.length())));
                return false;
            }
        }
        
        return true;
    }
    
    /**
     * 清理输入
     */
    public String sanitize(String input) {
        if (input == null) return null;
        
        // 移除潜在的危险字符
        String sanitized = input
            .replaceAll("[\\x00-\\x08\\x0B\\x0C\\x0E-\\x1F\\x7F]", "")
            .trim();
        
        // 限制长度
        if (sanitized.length() > 10000) {
            sanitized = sanitized.substring(0, 10000);
        }
        
        return sanitized;
    }
}

// 在 RAG 服务中使用
@Service
@RequiredArgsConstructor
public class RagService {
    
    private final PromptInjectionFilter injectionFilter;
    
    public RagResponse query(String userId, String question) {
        // 1. 输入清理
        String sanitizedQuestion = injectionFilter.sanitize(question);
        
        // 2. 注入检测
        if (!injectionFilter.isSafe(sanitizedQuestion)) {
            throw new SecurityException("检测到潜在的提示词注入攻击");
        }
        
        // 3. 正常处理
        // ...
    }
}
```

### 6.3 等保 2.0 合规检查表

| 控制项 | 要求 | RAG 系统实现 | 状态 |
|--------|------|-------------|------|
| 身份鉴别 | 双因素认证 | JWT + MFA | ✅ |
| 访问控制 | 权限最小化 | RBAC + 文档级权限 | ✅ |
| 安全审计 | 操作可追溯 | 全链路审计日志 | ✅ |
| 数据完整性 | 防篡改 | 日志签名 + 区块链存证 | ✅ |
| 数据保密性 | 加密存储 | AES-256 字段加密 | ✅ |
| 备份恢复 | 数据备份 | 每日备份 + 异地容灾 | ✅ |
| 入侵防范 | 攻击检测 | 高风险操作告警 | ✅ |

---

## 第 7 章 生产环境安全部署

### 7.1 安全配置清单

```yaml
# application-security.yml
security:
  jwt:
    secret: ${JWT_SECRET:}  # 必须从环境变量获取
    expiration: 3600000     # 1 小时
    refresh-expiration: 604800000  # 7 天
  
  encryption:
    aes-key: ${AES_KEY:}    # 必须从环境变量获取
  
  cors:
    allowed-origins: https://rag.yourdomain.com
    allow-credentials: true
  
  rate-limit:
    enabled: true
    requests-per-minute: 60
    burst-size: 10
  
  audit:
    enabled: true
    retention-days: 365
    high-risk-threshold: 4
```

### 7.2 Docker 安全配置

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine

# 创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# 复制 JAR
COPY target/rag-system.jar app.jar

# 设置权限
RUN chown -R appuser:appgroup /app

# 切换到非 root 用户
USER appuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 7.3 Nginx 安全配置

```nginx
# /etc/nginx/conf.d/rag-security.conf
server {
    listen 443 ssl http2;
    server_name rag.yourdomain.com;
    
    # SSL 配置
    ssl_certificate /etc/nginx/ssl/rag.crt;
    ssl_certificate_key /etc/nginx/ssl/rag.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline';" always;
    
    # 限流
    limit_req_zone $binary_remote_addr zone=rag_limit:10m rate=60r/m;
    
    location / {
        limit_req zone=rag_limit burst=10 nodelay;
        proxy_pass http://rag-frontend:80;
    }
    
    location /api/ {
        limit_req zone=rag_limit burst=10 nodelay;
        proxy_pass http://rag-backend:8080/;
        
        # IP 黑白名单
        allow 10.0.0.0/8;
        allow 192.168.0.0/16;
        deny all;
    }
}
```

---

**本专题完**

> 📌 **核心要点总结**：
> 1. RBAC + ABAC 混合权限模型满足企业级权限需求
> 2. 文档级权限控制需与向量检索过滤结合
> 3. 数据脱敏应在检索结果返回前执行
> 4. 审计日志需记录全链路操作，保留至少 365 天
> 5. 高风险操作（删除、导出、权限变更）需实时告警
> 6. 提示词注入防护是 LLM 应用特有的安全需求
> 7. JWT Token 需设置合理过期时间，支持刷新机制

> 📖 **下专题预告**：《RAG 系统运维与故障排查：从部署到监控的全流程指南》—— 详解容器编排、日志聚合、故障诊断、灾备恢复