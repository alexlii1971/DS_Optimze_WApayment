### **项目架构设计方案（第一部分）**  

---

#### **一、项目概述**  
**项目名称**：WordPress Multisite 微信 & 支付宝支付插件  
**核心目标**：  
1. 在 WordPress Multisite + WooCommerce 环境下，提供高效、安全的微信支付 & 支付宝支付功能。  
2. 支持网络级统一管理或子站独立管理，严格遵守 WordPress 和 WooCommerce 编码规范。  
3. 模块化设计，各功能解耦，支持单独启用/禁用。  

**适用场景**：  
- WordPress Multisite 网络，主站统一管理支付配置，子站继承规则。  
- 单站点独立激活时，子站管理员可独立配置支付参数。  

---

#### **二、架构设计原则**  
1. **遵循 WordPress & WooCommerce 规范**：  
   - 使用 `WP_Query`、`WC_Order` 等标准 API。  
   - 通过 WordPress Hook（Action/Filter）扩展功能，避免直接修改核心代码。  
2. **模块化设计**：  
   - 按功能划分独立模块（支付UI、Webhook、通知服务等），模块间通过事件驱动通信。  
   - 模块注册机制：通过 `PluginManager` 类动态加载模块。  
3. **网络级与子站级隔离**：  
   - 数据库表按 `site_id` 隔离（如日志、配置）。  
   - 网络级配置存储于 `wp_sitemeta`，子站配置存储于 `wp_options`。  
4. **安全性**：  
   - Webhook 请求需通过 HMAC-SHA256 签名验证。  
   - 支付回调 URL 动态生成，防止伪造请求。  

---

#### **三、模块划分与职责**  
| **模块名称**          | **职责**                                                                 | **依赖关系**              |  
|-----------------------|--------------------------------------------------------------------------|---------------------------|  
| **PaymentUIModule**   | 管理前端支付体验（倒计时、二维码刷新、自动跳转）                         | 无                        |  
| **NotificationModule** | 处理短信 & 邮件通知，支持网络级配置继承                                  | 依赖 `WC_Email` 类        |  
| **OrderProcessor**    | 处理订单支付 & 退款状态同步，存储 `site_id`，防重复退款                  | 依赖 `WC_Order`           |  
| **WebhookManager**    | Webhook 自动注册、失败重试、安全验证、日志记录                           | 依赖 `WC_Webhook`         |  
| **AdminModule**       | 提供网络级 & 子站级管理界面（支付配置、日志查看、统计）                  | 依赖 WordPress Admin API  |  
| **APIClient**         | 封装微信 & 支付宝 API 调用（统一下单、查询支付状态）                     | 无                        |  

---

#### **四、技术实现路径**  

##### **1. 支付回调地址动态生成**  
**问题**：多站点环境下，子站需独立回调地址以匹配订单。  
**实现方案**：  
- **主站回调地址**：`https://tao.ooo/wc-api/wechat_callback`  
- **子站回调地址**：`https://sub.tao.ooo/wc-api/wechat_callback`  
- **动态生成逻辑**：  
  ```php  
  function generate_callback_url($type) {  
      $blog_id = get_current_blog_id();  
      $base_url = get_site_url($blog_id);  
      return $base_url . "/wc-api/{$type}_callback";  
  }  
  ```  

##### **2. 订单存储 `site_id`**  
**数据库设计**：  
- 扩展 `wp_posts` 表，通过 `_site_id` Meta 字段标记订单所属子站。  
- **Hook 实现**：  
  ```php  
  add_action('woocommerce_checkout_create_order', function ($order) {  
      $order->update_meta_data('_site_id', get_current_blog_id());  
  });  
  ```  

##### **3. Webhook 自动注册与维护**  
**实现步骤**：  
1. **插件激活时**：遍历所有子站，为每个站点创建独立 Webhook。  
2. **子站创建时**：通过 `wpmu_new_blog` Hook 自动注册 Webhook。  
3. **Webhook 参数**：  
   ```php  
   $webhook = new WC_Webhook();  
   $webhook->set_name('微信支付回调');  
   $webhook->set_topic('payment.wechat');  
   $webhook->set_delivery_url(generate_callback_url('wechat'));  
   $webhook->set_secret(wp_generate_password(32));  
   ```  

##### **4. 网络级与子站级管理地址**  
- **网络级管理界面**：  
  URL: `https://tao.ooo/wp-admin/network/admin.php?page=wc-multisite-payment`  
  功能：配置全局支付网关、强制启用短信通知、查看全站支付日志。  
- **子站级管理界面**：  
  URL: `https://sub.tao.ooo/wp-admin/admin.php?page=wc-payment-settings`  
  功能：子站独立配置支付 API、查看本地日志、手动同步订单状态。  

---

#### **五、优化分析与调整**  

##### **优化点 1：模块通信解耦**  
**问题**：原方案中 `WebhookManager` 直接调用 `OrderProcessor` 的方法，导致耦合。  
**优化方案**：  
- 使用 WordPress 的 `do_action` 实现事件驱动：  
  ```php  
  // Webhook 接收到支付成功事件时触发  
  do_action('wc_payment_success', $order_id, $transaction_id);  
  // OrderProcessor 监听该事件  
  add_action('wc_payment_success', [$this, 'update_order_status']);  
  ```  

##### **优化点 2：支付配置继承机制**  
**问题**：子站无法灵活覆盖网络级配置（如短信 API 密钥）。  
**优化方案**：  
- 优先级逻辑：子站配置 > 网络级配置 > 默认值。  
- **代码实现**：  
  ```php  
  function get_sms_api_key() {  
      $local_key = get_option('sms_api_key');  
      $network_key = get_site_option('sms_api_key');  
      return $local_key ?: $network_key;  
  }  
  ```  

---

### **项目架构设计方案（第二部分）**  

---

#### **六、关键代码结构**  

##### **1. 模块注册与加载**  
**核心类：`PluginManager`**  
```php  
class PluginManager {  
    private $modules = [];  

    public function register_module($module) {  
        $this->modules[] = $module;  
    }  

    public function init() {  
        foreach ($this->modules as $module) {  
            if ($module->is_enabled()) {  
                $module->init();  
            }  
        }  
    }  
}  

// 初始化示例  
$manager = new PluginManager();  
$manager->register_module(new PaymentUIModule());  
$manager->register_module(new WebhookManager());  
$manager->init();  
```  

##### **2. 支付状态同步（API 查询）**  
**类：`APIClient`**  
```php  
class APIClient {  
    public function query_payment_status($order_id) {  
        $site_id = get_post_meta($order_id, '_site_id', true);  
        switch_to_blog($site_id);  

        $api_key = get_option('wechat_api_key');  
        // 调用微信/支付宝 API  
        $status = $this->call_wechat_api($order_id, $api_key);  

        restore_current_blog();  
        return $status;  
    }  
}  
```  

##### **3. Webhook 安全验证**  
**类：`WebhookSecurity`**  
```php  
class WebhookSecurity {  
    public function verify_signature($payload) {  
        $secret = get_option('webhook_secret');  
        $signature = hash_hmac('sha256', json_encode($payload), $secret);  
        return $signature === $_SERVER['HTTP_X_SIGNATURE'];  
    }  

    public function prevent_replay_attack($timestamp) {  
        $current_time = time();  
        return abs($current_time - $timestamp) < 300; // 5 分钟有效期  
    }  
}  
```  

---

#### **七、部署与兼容性**  

##### **1. 多站点兼容性**  
- **子站独立域名支持**：  
  ```php  
  add_filter('site_url', function ($url, $path) {  
      $blog_id = get_current_blog_id();  
      if (is_subdomain_install()) {  
          $domain = get_blog_details($blog_id)->domain;  
          return "https://{$domain}{$path}";  
      }  
      return $url;  
  }, 10, 2);  
  ```  

##### **2. 数据库表设计**  
| **表名**               | **字段**                  | **说明**                          |  
|-------------------------|---------------------------|-----------------------------------|  
| `wp_webhook_failures`   | `id`, `endpoint`, `retry_count`, `last_attempt` | Webhook 失败记录                |  
| `wp_payment_logs`       | `order_id`, `site_id`, `status`, `amount`       | 支付日志（按 `site_id` 隔离）   |  

---

#### **八、未来扩展方向**  
1. **事件驱动架构增强**：  
   - 使用 `WP_Queue` 实现异步任务处理（如延迟重试 Webhook）。  
2. **支付网关抽象层**：  
   - 定义 `PaymentGateway` 接口，便于扩展 Stripe/PayPal。  
3. **自动化测试覆盖**：  
   - 基于 PHPUnit 构建测试用例，覆盖核心支付流程。  

---
### **项目架构设计方案（第三部分）**  

---

#### **九、数据库与数据隔离设计**  

##### **1. 数据存储策略**  
- **网络级配置**：使用 `wp_sitemeta` 表存储全局支付参数（如默认支付方式、强制启用的功能）。  
- **子站级配置**：使用 `wp_options` 表存储子站独立配置（如微信/支付宝 API 密钥、短信通知开关）。  
- **订单数据隔离**：通过 `_site_id` Meta 字段标记订单所属子站，确保回调时精准匹配。  

##### **2. 日志隔离设计**  
- **支付日志表**：`wp_{blog_id}_payment_logs`  
  - 字段：`log_id`, `order_id`, `action`（支付/退款）, `status`, `timestamp`  
  - 按子站分表存储，避免全站日志混杂。  
- **Webhook 日志表**：`wp_webhook_logs`  
  - 字段：`webhook_id`, `endpoint`, `payload`, `response_code`, `site_id`  
  - 通过 `site_id` 字段区分不同子站的 Webhook 请求。  

##### **3. 分表查询优化**  
- 使用 WordPress 的 `switch_to_blog()` 函数动态切换数据库上下文：  
  ```php  
  function get_payment_logs($blog_id) {  
      switch_to_blog($blog_id);  
      $logs = $wpdb->get_results("SELECT * FROM {$wpdb->prefix}payment_logs");  
      restore_current_blog();  
      return $logs;  
  }  
  ```  

---

#### **十、安全性增强设计**  

##### **1. 支付请求防篡改**  
- **签名机制**：  
  - 前端提交支付请求时，生成 `nonce` 和 `signature`：  
    ```php  
    $nonce = wp_generate_password(12);  
    $signature = hash_hmac('sha256', $order_id . $nonce, SECRET_KEY);  
    ```  
  - 后端验证签名有效性，防止参数篡改。  

##### **2. Webhook 安全防护**  
- **IP 白名单**：仅允许微信/支付宝官方 IP 段触发 Webhook。  
- **重放攻击防御**：  
  - 在 Webhook 请求中携带时间戳，服务器端校验时间差（±5 分钟）。  
  ```php  
  if (abs(time() - $timestamp) > 300) {  
      wp_die("请求已过期", 403);  
  }  
  ```  

##### **3. 敏感数据加密**  
- 使用 WordPress 的 `wp_salt()` 动态生成加密密钥。  
- 支付 API 密钥存储时加密：  
  ```php  
  update_option('wechat_api_key', openssl_encrypt($raw_key, 'AES-256-CBC', wp_salt()));  
  ```  

---

#### **十一、网络级与子站级管理界面实现**  

##### **1. 网络级管理界面**  
- **URL**: `https://tao.ooo/wp-admin/network/admin.php?page=wc-multisite-payment`  
- **功能**：  
  - 配置全局支付方式（强制启用微信/支付宝）。  
  - 设置短信通知模板，并强制子站继承。  
  - 查看全站支付成功率、退款率统计图表。  
  - 管理 Webhook 全局规则（如重试次数、失败警报阈值）。  

##### **2. 子站级管理界面**  
- **URL**: `https://sub.tao.ooo/wp-admin/admin.php?page=wc-payment-settings`  
- **功能**：  
  - 独立配置微信/支付宝 API 参数（商户号、密钥）。  
  - 启用/禁用短信通知（若网络级未强制启用）。  
  - 手动同步订单支付状态、查看本地支付日志。  

##### **3. 权限控制**  
- **网络管理员**：可修改所有子站配置，强制功能继承。  
- **子站管理员**：仅能修改本地配置，无权访问其他子站数据。  
- **代码实现**：  
  ```php  
  add_action('admin_init', function() {  
      if (is_network_admin() && !current_user_can('manage_network')) {  
          wp_die("无权访问网络级设置");  
      }  
  });  
  ```  

---

#### **十二、异常处理与容灾机制**  

##### **1. 支付流程异常处理**  
- **二维码过期**：前端定时轮询接口，自动刷新二维码（间隔 30 秒）。  
- **支付超时**：订单状态标记为“已取消”，释放库存并通知用户。  

##### **2. Webhook 失败容灾**  
- **自动降级**：若 Webhook 连续失败，自动切换至 API 主动查询模式。  
- **人工干预通道**：在管理界面提供“紧急同步”按钮，手动触发全站订单状态同步。  

##### **3. 日志与监控**  
- **ELK 集成**：支付日志推送至 Elasticsearch，实现实时监控与告警。  
- **关键指标监控**：  
  - 支付成功率 < 95% 时触发邮件告警。  
  - Webhook 失败率 > 5% 时自动暂停并通知管理员。  

---

#### **十三、性能优化方案**  

##### **1. 静态资源缓存**  
- 支付页面 JS/CSS 文件添加版本号，利用浏览器缓存：  
  ```php  
  wp_enqueue_script('payment-js', plugins_url('js/payment.js', __FILE__), [], filemtime(plugin_dir_path(__FILE__) . 'js/payment.js'));  
  ```  

##### **2. 数据库查询优化**  
- 订单查询时强制指定 `site_id`，避免全表扫描：  
  ```php  
  $orders = $wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_type = 'shop_order' AND ID IN (SELECT post_id FROM {$wpdb->postmeta} WHERE meta_key = '_site_id' AND meta_value = $current_blog_id)");  
  ```  

##### **3. 异步任务队列**  
- 使用 `WP_Background_Process` 类处理耗时操作（如批量同步订单状态）：  
  ```php  
  class Payment_Sync_Process extends WP_Background_Process {  
      protected function task($order_id) {  
          // 调用 API 查询并更新订单状态  
          return false;  
      }  
  }  
  ```  

---

#### **十四、测试与部署流程**  

##### **1. 单元测试覆盖**  
- **PHPUnit 测试用例**：  
  - 支付回调正确性验证。  
  - Webhook 签名校验逻辑测试。  
  - 多站点数据隔离测试。  

##### **2. 灰度发布策略**  
- 首次部署至测试子站（如 `test.tao.ooo`），验证支付全流程。  
- 逐步开放至 10% -> 50% -> 100% 子站，监控错误日志。  

##### **3. 回滚方案**  
- 保留旧版本插件压缩包，出现严重问题时通过 WordPress 后台一键回滚。  

---

#### **十五、最终架构总结**  

##### **架构优势**  
1. **严格遵循规范**：完全基于 WordPress Hook 和 WooCommerce API，无侵入式修改。  
2. **高性能与安全**：通过异步队列、数据加密、IP 白名单等机制保障系统稳定。  
3. **灵活扩展**：模块化设计允许未来新增支付方式（如 Stripe）或功能（如订阅支付）。  

##### **待优化点**  
- **支付风控模块**：当前未集成风险控制（如同一 IP 频繁支付），需后续扩展。  
- **国际化支持**：UI 文本未完全适配多语言，可通过 `wp-i18n` 优化。  

---
