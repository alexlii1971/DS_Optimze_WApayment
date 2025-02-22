### **WordPress Multisite WooCommerce 微信支付宝支付插件开发方案**

---

#### **一、项目概述**
本插件旨在为 **WordPress Multisite** 环境下的 **WooCommerce** 提供微信支付和支付宝支付功能，支持多站点独立配置和网络级统一管理，满足不同场景的支付需求。插件需支持最新版本的 WordPress 和 WooCommerce，并充分考虑多站点环境下的复杂场景，如子站点绑定独立域名、网络级统一管理等。

---

### **二、核心功能需求**

#### **1. 支持 WordPress Multisite 与 WooCommerce**
- **WordPress 版本**：支持最新版本（6.7.2+），并兼容 Multisite 模式。
- **WooCommerce 版本**：支持最新版本（9.6.2+）。
- **多站点支持**：
  - 支持子站点绑定独立域名。
  - 插件需在不同场景下（二级域名和独立域名）正常运行。

#### **2. 插件激活方式**
- **子站单独激活**：
  - 支持子站点独立配置支付参数（商户信息、SDK、证书、App ID、Secret、回调地址等）。
  - 提供子站点独立的管理界面。
- **网络全站激活**：
  - 支持网络级统一管理支付配置。
  - 子站点可继承主站点的支付配置。
  - 提供网络级管理面板（支付、统计）。

#### **3. 支付功能**
- **微信支付**：
  - 手机网页端唤醒微信 App 支付。
  - 电脑端扫码支付。
  - 兼容微信登录插件的 OpenID 或 UnionID。
- **支付宝支付**：
  - 手机网页端唤醒支付宝 App 支付。
  - 电脑端扫码支付。

#### **4. 配置管理**
- 提供可视化的配置管理界面。
- 支持微信支付和支付宝支付的相关参数配置（证书、API、Key、URL 等）。
- 提供详细的配置说明。

#### **5. 模块化设计**
- 功能模块化，避免过度耦合。
- 提供完善的管理、配置和文档。

#### **6. 日志功能**
- 支持日志记录，可在 WooCommerce 状态中查看日志。
- 提供“启用日志”选项按钮。

#### **7. 编码规范**
- 遵循 WordPress 和 WooCommerce 的 API 规则和最佳实践。
- 插件激活时检查 WooCommerce 是否已激活。

---

### **三、技术实现方案**

#### **1. 多站点配置管理**
```php
class Multisite_Payment_Config {
    // 获取网络级配置
    public function get_network_config($key) {
        return get_site_option("wcpay_network_{$key}");
    }

    // 获取站点级配置（带继承逻辑）
    public function get_blog_config($blog_id, $key) {
        $value = get_blog_option($blog_id, "wcpay_blog_{$key}");
        return $value ?: $this->get_network_config($key);
    }

    // 配置界面渲染
    public function render_settings() {
        woocommerce_admin_fields($this->get_settings());
    }
}
```

#### **2. 微信支付适配器**
```php
class WC_WeChat_Gateway extends WC_Payment_Gateway {
    // 支付请求生成
    public function process_payment($order_id) {
        $order = wc_get_order($order_id);
        $openid = get_user_meta($order->get_user_id(), 'wechat_openid', true);
        
        $params = [
            'body' => $this->get_order_title($order),
            'out_trade_no' => $this->get_trade_no($order),
            'total_fee' => $order->get_total() * 100,
            'openid' => $openid,
            'notify_url' => $this->notify_url
        ];

        $response = WeChat_SDK::create_order($params);
        
        if ($response->is_success()) {
            return [
                'result' => 'success',
                'wechat_data' => $response->get_jsapi_params()
            ];
        }
    }
    
    // OpenID 校验
    private function validate_openid($openid) {
        return preg_match('/^o[A-Za-z0-9_-]{28}$/', $openid);
    }
}
```

#### **3. 支付宝异步通知处理**
```php
add_action('woocommerce_api_wc_gateway_alipay', function() {
    $raw_data = file_get_contents('php://input');
    $data = json_decode($raw_data, true);

    // 验证签名
    if (!Alipay_SDK::verify_signature($data)) {
        wp_send_json_error('Invalid signature', 403);
    }

    $order = wc_get_order($data['out_trade_no']);
    
    // 处理不同交易状态
    switch ($data['trade_status']) {
        case 'TRADE_SUCCESS':
            $order->payment_complete();
            break;
        case 'TRADE_CLOSED':
            $order->update_status('cancelled');
            break;
    }

    echo 'success'; // 必须返回 success 确认通知
});
```

#### **4. 日志功能**
```php
class Payment_Logger {
    public function log($message, $level = 'info') {
        if (get_option('wcpay_enable_log')) {
            $log_entry = sprintf("[%s] %s: %s\n", date('Y-m-d H:i:s'), strtoupper($level), $message);
            file_put_contents(WC_LOG_DIR . 'wcpay-payment.log', $log_entry, FILE_APPEND);
        }
    }
}
```

---

### **四、安全增强措施**

#### **1. 密钥管理**
```php
class Key_Manager {
    const CIPHER_METHOD = 'aes-256-cbc';
    
    public function encrypt($plaintext) {
        $iv = substr(SECURE_AUTH_SALT, 0, 16);
        return openssl_encrypt(
            $plaintext,
            self::CIPHER_METHOD,
            SECURE_AUTH_KEY,
            0,
            $iv
        );
    }

    public function decrypt($ciphertext) {
        $iv = substr(SECURE_AUTH_SALT, 0, 16);
        return openssl_decrypt(
            $ciphertext,
            self::CIPHER_METHOD,
            SECURE_AUTH_KEY,
            0,
            $iv
        );
    }
}
```

#### **2. 防重放攻击**
```php
class Replay_Protection {
    private $nonce_cache;

    public function __construct() {
        $this->nonce_cache = new WP_Object_Cache();
    }

    public function verify_nonce($nonce) {
        if ($this->nonce_cache->get($nonce)) {
            return false; // 重复请求
        }
        $this->nonce_cache->set($nonce, true, 300); // 缓存 5 分钟
        return true;
    }
}
```

---

### **五、测试与验收**

#### **1. 功能测试矩阵**
| **测试场景**        | **预期结果**                | **验收标准**           |
|---------------------|----------------------------|------------------------|
| 微信 H5 支付        | 成功跳转微信完成支付        | 订单状态变更为完成     |
| 支付宝扫码支付      | 生成有效二维码              | 扫码后成功付款         |
| 多站点独立配置      | 各站点显示正确支付方式      | 配置互不影响           |
| 支付中断恢复        | 保留待支付订单              | 可重新发起支付         |

#### **2. 性能基准要求**
| **类型**         | **单节点 TPS** | **集群 TPS** | **响应时间** |
|------------------|----------------|--------------|--------------|
| 微信支付         | 300            | 1500         | <800ms       |
| 支付宝支付       | 500            | 2500         | <500ms       |
| 混合支付         | 200            | 1000         | <1s          |

---

### **六、部署与维护**

#### **1. 部署流程**
```bash
# 安装插件
wp plugin install wcpay-wechat-alipay --activate-network

# 配置支付参数
wp option set wcpay_wechat_appid "your_appid" --url=example.com
wp option set wcpay_alipay_appid "your_appid" --url=example.com
```

#### **2. 维护策略**
- **版本支持**：
  - LTS 版本支持 24 个月。
  - 常规版本支持 12 个月。
- **更新机制**：
  - 自动更新签名验证。
  - 灰度发布策略（10% -> 50% -> 100%）。

---

### **七、总结**
本插件通过模块化设计和多站点支持，为 WordPress Multisite 环境下的 WooCommerce 提供了完整的微信支付和支付宝支付解决方案。通过严格的测试和安全措施，确保插件的高可用性和安全性。


感谢您的提醒！确实，在之前的方案中忽略了“模式二”的需求，即 **全站网络激活并支持分账功能** 的场景。以下是对“模式二”需求的详细补充和完整方案更新。

---

### **八、模式二：全站网络激活并支持分账功能**

#### **1. 需求描述**
- **场景**：平台级支付、收款、分账方案。
- **功能**：
  - 支持微信支付、支付宝支付的分账功能。
  - 支持卖家后台提交申请特约商户，供微信、支付宝合规审核。
  - 配置与管理分账功能。
  - 管理特约商户的订单合规性，处理可能的违规退款。

#### **2. 技术实现方案**

##### **2.1 分账功能设计**
```php
class Split_Payment_Manager {
    // 分账规则配置
    public function set_split_rule($rule) {
        update_option('wcpay_split_rule', $rule);
    }

    // 执行分账
    public function execute_split($order_id) {
        $order = wc_get_order($order_id);
        $split_rule = get_option('wcpay_split_rule');
        
        foreach ($split_rule as $rule) {
            $amount = $order->get_total() * $rule['percentage'];
            $this->transfer_to_merchant($rule['merchant_id'], $amount);
        }
    }

    // 转账到特约商户
    private function transfer_to_merchant($merchant_id, $amount) {
        // 调用支付平台 API 执行分账
        $response = WeChat_SDK::transfer($merchant_id, $amount);
        if (!$response->is_success()) {
            $this->log("分账失败: 商户 {$merchant_id}, 金额 {$amount}");
        }
    }
}
```

##### **2.2 特约商户管理**
```php
class Sub_Merchant_Manager {
    // 提交特约商户申请
    public function submit_application($merchant_data) {
        $response = WeChat_SDK::create_sub_merchant($merchant_data);
        if ($response->is_success()) {
            $this->save_merchant_info($response->get_data());
        }
        return $response;
    }

    // 保存商户信息
    private function save_merchant_info($data) {
        update_option('wcpay_sub_merchant_' . $data['merchant_id'], $data);
    }

    // 获取商户状态
    public function get_merchant_status($merchant_id) {
        $merchant_data = get_option('wcpay_sub_merchant_' . $merchant_id);
        return $merchant_data['status'] ?? 'unknown';
    }
}
```

##### **2.3 订单合规性管理**
```php
class Order_Compliance_Manager {
    // 检查订单合规性
    public function check_compliance($order_id) {
        $order = wc_get_order($order_id);
        $compliance_rules = get_option('wcpay_compliance_rules');

        foreach ($compliance_rules as $rule) {
            if (!$this->validate_rule($order, $rule)) {
                $this->mark_as_non_compliant($order_id);
                break;
            }
        }
    }

    // 验证规则
    private function validate_rule($order, $rule) {
        // 示例：检查订单金额是否超过限额
        if ($rule['type'] === 'amount_limit' && $order->get_total() > $rule['value']) {
            return false;
        }
        return true;
    }

    // 标记为不合规
    private function mark_as_non_compliant($order_id) {
        update_post_meta($order_id, '_compliance_status', 'non_compliant');
    }
}
```

##### **2.4 违规退款处理**
```php
class Refund_Manager {
    // 处理违规退款
    public function process_refund($order_id) {
        $order = wc_get_order($order_id);
        $refund_amount = $order->get_total();

        // 调用支付平台 API 执行退款
        $response = WeChat_SDK::refund($order->get_transaction_id(), $refund_amount);
        if ($response->is_success()) {
            $order->update_status('refunded');
        }
    }
}
```

---

### **九、模式二的功能测试**

#### **1. 分账功能测试**
| **测试场景**            | **预期结果**                | **验收标准**           |
|-------------------------|----------------------------|------------------------|
| 分账规则配置            | 分账规则保存成功            | 规则可正常查询         |
| 执行分账                | 分账金额正确分配到商户      | 分账记录可查           |
| 分账失败处理            | 分账失败记录日志            | 日志中记录失败信息     |

#### **2. 特约商户管理测试**
| **测试场景**            | **预期结果**                | **验收标准**           |
|-------------------------|----------------------------|------------------------|
| 提交特约商户申请        | 申请提交成功                | 返回成功状态           |
| 获取商户状态            | 返回商户当前状态            | 状态信息准确           |

#### **3. 订单合规性测试**
| **测试场景**            | **预期结果**                | **验收标准**           |
|-------------------------|----------------------------|------------------------|
| 检查订单合规性          | 不合规订单被标记            | 订单状态更新           |
| 违规退款处理            | 退款成功且订单状态更新      | 退款记录可查           |

---

### **十、模式二的部署与维护**

#### **1. 部署流程**
```bash
# 启用分账功能
wp option set wcpay_enable_split_payment true --network

# 配置分账规则
wp option set wcpay_split_rule '{"merchant_id_1": 0.7, "merchant_id_2": 0.3}' --network
```

#### **2. 维护策略**
- **分账规则更新**：支持动态更新分账比例。
- **商户状态监控**：定期检查特约商户的状态，确保合规性。
- **退款处理**：自动处理不合规订单的退款。

---

### **十一、总结**
通过补充“模式二”的需求，插件不仅支持基础的支付功能，还提供了强大的分账和合规管理能力，适用于平台级支付场景。以下是本方案的完整功能列表：

1. **多站点支持**：
   - 子站点独立配置。
   - 网络级统一管理。
2. **支付功能**：
   - 微信支付（H5、扫码）。
   - 支付宝支付（H5、扫码）。
3. **分账功能**：
   - 分账规则配置。
   - 特约商户管理。
   - 订单合规性检查。
   - 违规退款处理。
4. **安全与日志**：
   - 密钥管理。
   - 防重放攻击。
   - 日志记录与监控。

通过严格的测试和灵活的部署方案，本插件能够满足不同规模站点的需求，确保支付功能的高效、安全和合规。


### **WordPress Multisite WooCommerce 微信支付宝支付插件开发方案**

---

#### **一、项目概述**
本插件旨在为 **WordPress Multisite** 环境下的 **WooCommerce** 提供微信支付和支付宝支付功能，支持多站点独立配置和网络级统一管理，满足不同场景的支付需求。插件需支持最新版本的 WordPress 和 WooCommerce，并充分考虑多站点环境下的复杂场景，如子站点绑定独立域名、网络级统一管理、分账功能等。

---

### **二、核心功能需求**

#### **1. 支持 WordPress Multisite 与 WooCommerce**
- **WordPress 版本**：支持最新版本（6.7.2+），并兼容 Multisite 模式。
- **WooCommerce 版本**：支持最新版本（9.6.2+）。
- **多站点支持**：
  - 支持子站点绑定独立域名。
  - 插件需在不同场景下（二级域名和独立域名）正常运行。

#### **2. 插件激活方式**
- **子站单独激活**：
  - 支持子站点独立配置支付参数（商户信息、SDK、证书、App ID、Secret、回调地址等）。
  - 提供子站点独立的管理界面。
- **网络全站激活**：
  - 支持网络级统一管理支付配置。
  - 子站点可继承主站点的支付配置。
  - 提供网络级管理面板（支付、统计）。

#### **3. 支付功能**
- **微信支付**：
  - 手机网页端唤醒微信 App 支付。
  - 电脑端扫码支付。
  - 兼容微信登录插件的 OpenID 或 UnionID。
- **支付宝支付**：
  - 手机网页端唤醒支付宝 App 支付。
  - 电脑端扫码支付。

#### **4. 分账功能（模式二）**
- **场景**：平台级支付、收款、分账方案。
- **功能**：
  - 支持微信支付、支付宝支付的分账功能。
  - 支持卖家后台提交申请特约商户，供微信、支付宝合规审核。
  - 配置与管理分账功能。
  - 管理特约商户的订单合规性，处理可能的违规退款。

#### **5. 配置管理**
- 提供可视化的配置管理界面。
- 支持微信支付和支付宝支付的相关参数配置（证书、API、Key、URL 等）。
- 提供详细的配置说明。

#### **6. 模块化设计**
- 功能模块化，避免过度耦合。
- 提供完善的管理、配置和文档。

#### **7. 日志功能**
- 支持日志记录，可在 WooCommerce 状态中查看日志。
- 提供“启用日志”选项按钮。

#### **8. 编码规范**
- 遵循 WordPress 和 WooCommerce 的 API 规则和最佳实践。
- 插件激活时检查 WooCommerce 是否已激活。

---

### **三、技术实现方案**

#### **1. 多站点配置管理**
```php
class Multisite_Payment_Config {
    // 获取网络级配置
    public function get_network_config($key) {
        return get_site_option("wcpay_network_{$key}");
    }

    // 获取站点级配置（带继承逻辑）
    public function get_blog_config($blog_id, $key) {
        $value = get_blog_option($blog_id, "wcpay_blog_{$key}");
        return $value ?: $this->get_network_config($key);
    }

    // 配置界面渲染
    public function render_settings() {
        woocommerce_admin_fields($this->get_settings());
    }
}
```

#### **2. 微信支付适配器**
```php
class WC_WeChat_Gateway extends WC_Payment_Gateway {
    // 支付请求生成
    public function process_payment($order_id) {
        $order = wc_get_order($order_id);
        $openid = get_user_meta($order->get_user_id(), 'wechat_openid', true);
        
        $params = [
            'body' => $this->get_order_title($order),
            'out_trade_no' => $this->get_trade_no($order),
            'total_fee' => $order->get_total() * 100,
            'openid' => $openid,
            'notify_url' => $this->notify_url
        ];

        $response = WeChat_SDK::create_order($params);
        
        if ($response->is_success()) {
            return [
                'result' => 'success',
                'wechat_data' => $response->get_jsapi_params()
            ];
        }
    }
    
    // OpenID 校验
    private function validate_openid($openid) {
        return preg_match('/^o[A-Za-z0-9_-]{28}$/', $openid);
    }
}
```

#### **3. 支付宝异步通知处理**
```php
add_action('woocommerce_api_wc_gateway_alipay', function() {
    $raw_data = file_get_contents('php://input');
    $data = json_decode($raw_data, true);

    // 验证签名
    if (!Alipay_SDK::verify_signature($data)) {
        wp_send_json_error('Invalid signature', 403);
    }

    $order = wc_get_order($data['out_trade_no']);
    
    // 处理不同交易状态
    switch ($data['trade_status']) {
        case 'TRADE_SUCCESS':
            $order->payment_complete();
            break;
        case 'TRADE_CLOSED':
            $order->update_status('cancelled');
            break;
    }

    echo 'success'; // 必须返回 success 确认通知
});
```

#### **4. 分账功能实现**
```php
class Split_Payment_Manager {
    // 分账规则配置
    public function set_split_rule($rule) {
        update_option('wcpay_split_rule', $rule);
    }

    // 执行分账
    public function execute_split($order_id) {
        $order = wc_get_order($order_id);
        $split_rule = get_option('wcpay_split_rule');
        
        foreach ($split_rule as $rule) {
            $amount = $order->get_total() * $rule['percentage'];
            $this->transfer_to_merchant($rule['merchant_id'], $amount);
        }
    }

    // 转账到特约商户
    private function transfer_to_merchant($merchant_id, $amount) {
        // 调用支付平台 API 执行分账
        $response = WeChat_SDK::transfer($merchant_id, $amount);
        if (!$response->is_success()) {
            $this->log("分账失败: 商户 {$merchant_id}, 金额 {$amount}");
        }
    }
}
```

#### **5. 特约商户管理**
```php
class Sub_Merchant_Manager {
    // 提交特约商户申请
    public function submit_application($merchant_data) {
        $response = WeChat_SDK::create_sub_merchant($merchant_data);
        if ($response->is_success()) {
            $this->save_merchant_info($response->get_data());
        }
        return $response;
    }

    // 保存商户信息
    private function save_merchant_info($data) {
        update_option('wcpay_sub_merchant_' . $data['merchant_id'], $data);
    }

    // 获取商户状态
    public function get_merchant_status($merchant_id) {
        $merchant_data = get_option('wcpay_sub_merchant_' . $merchant_id);
        return $merchant_data['status'] ?? 'unknown';
    }
}
```

#### **6. 订单合规性管理**
```php
class Order_Compliance_Manager {
    // 检查订单合规性
    public function check_compliance($order_id) {
        $order = wc_get_order($order_id);
        $compliance_rules = get_option('wcpay_compliance_rules');

        foreach ($compliance_rules as $rule) {
            if (!$this->validate_rule($order, $rule)) {
                $this->mark_as_non_compliant($order_id);
                break;
            }
        }
    }

    // 验证规则
    private function validate_rule($order, $rule) {
        // 示例：检查订单金额是否超过限额
        if ($rule['type'] === 'amount_limit' && $order->get_total() > $rule['value']) {
            return false;
        }
        return true;
    }

    // 标记为不合规
    private function mark_as_non_compliant($order_id) {
        update_post_meta($order_id, '_compliance_status', 'non_compliant');
    }
}
```

#### **7. 违规退款处理**
```php
class Refund_Manager {
    // 处理违规退款
    public function process_refund($order_id) {
        $order = wc_get_order($order_id);
        $refund_amount = $order->get_total();

        // 调用支付平台 API 执行退款
        $response = WeChat_SDK::refund($order->get_transaction_id(), $refund_amount);
        if ($response->is_success()) {
            $order->update_status('refunded');
        }
    }
}
```

#### **8. 日志功能**
```php
class Payment_Logger {
    public function log($message, $level = 'info') {
        if (get_option('wcpay_enable_log')) {
            $log_entry = sprintf("[%s] %s: %s\n", date('Y-m-d H:i:s'), strtoupper($level), $message);
            file_put_contents(WC_LOG_DIR . 'wcpay-payment.log', $log_entry, FILE_APPEND);
        }
    }
}
```

---

### **四、安全增强措施**

#### **1. 密钥管理**
```php
class Key_Manager {
    const CIPHER_METHOD = 'aes-256-cbc';
    
    public function encrypt($plaintext) {
        $iv = substr(SECURE_AUTH_SALT, 0, 16);
        return openssl_encrypt(
            $plaintext,
            self::CIPHER_METHOD,
            SECURE_AUTH_KEY,
            0,
            $iv
        );
    }

    public function decrypt($ciphertext) {
        $iv = substr(SECURE_AUTH_SALT, 0, 16);
        return openssl_decrypt(
            $ciphertext,
            self::CIPHER_METHOD,
            SECURE_AUTH_KEY,
            0,
            $iv
        );
    }
}
```

#### **2. 防重放攻击**
```php
class Replay_Protection {
    private $nonce_cache;

    public function __construct() {
        $this->nonce_cache = new WP_Object_Cache();
    }

    public function verify_nonce($nonce) {
        if ($this->nonce_cache->get($nonce)) {
            return false; // 重复请求
        }
        $this->nonce_cache->set($nonce, true, 300); // 缓存 5 分钟
        return true;
    }
}
```

---

### **五、测试与验收**

#### **1. 功能测试矩阵**
| **测试场景**            | **预期结果**                | **验收标准**           |
|-------------------------|----------------------------|------------------------|
| 微信 H5 支付            | 成功跳转微信完成支付        | 订单状态变更为完成     |
| 支付宝扫码支付          | 生成有效二维码              | 扫码后成功付款         |
| 多站点独立配置          | 各站点显示正确支付方式      | 配置互不影响           |
| 支付中断恢复            | 保留待支付订单              | 可重新发起支付         |
| 分账规则配置            | 分账规则保存成功            | 规则可正常查询         |
| 执行分账                | 分账金额正确分配到商户      | 分账记录可查           |
| 分账失败处理            | 分账失败记录日志            | 日志中记录失败信息     |
| 提交特约商户申请        | 申请提交成功                | 返回成功状态           |
| 获取商户状态            | 返回商户当前状态            | 状态信息准确           |
| 检查订单合规性          | 不合规订单被标记            | 订单状态更新           |
| 违规退款处理            | 退款成功且订单状态更新      | 退款记录可查           |

#### **2. 性能基准要求**
| **类型**         | **单节点 TPS** | **集群 TPS** | **响应时间** |
|------------------|----------------|--------------|--------------|
| 微信支付         | 300            | 1500         | <800ms       |
| 支付宝支付       | 500            | 2500         | <500ms       |
| 混合支付         | 200            | 1000         | <1s          |

---

### **六、部署与维护**

#### **1. 部署流程**
```bash
# 安装插件
wp plugin install wcpay-wechat-alipay --activate-network

# 配置支付参数
wp option set wcpay_wechat_appid "your_appid" --url=example.com
wp option set wcpay_alipay_appid "your_appid" --url=example.com

# 启用分账功能
wp option set wcpay_enable_split_payment true --network

# 配置分账规则
wp option set wcpay_split_rule '{"merchant_id_1": 0.7, "merchant_id_2": 0.3}' --network
```

#### **2. 维护策略**
- **版本支持**：
  - LTS 版本支持 24 个月。
  - 常规版本支持 12 个月。
- **更新机制**：
  - 自动更新签名验证。
  - 灰度发布策略（10% -> 50% -> 100%）。

---

### **七、总结**
本插件通过模块化设计和多站点支持，为 WordPress Multisite 环境下的 WooCommerce 提供了完整的微信支付和支付宝支付解决方案，同时支持分账功能和合规管理。通过严格的测试和安全措施，确保插件的高可用性和安全性，适用于从单站点到平台级的多场景支付需求。
