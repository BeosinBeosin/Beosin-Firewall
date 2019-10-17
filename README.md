# 防火墙文档

## 功能说明
1. 依托Beosin成都链安准确的风险检测引擎和丰富的恶意地址库，严格过滤来自黑客攻击、羊毛党、钓鱼、诈骗等的恶意交易行为。
2. 用户可设置自定义安全策略，目前包括自定义黑白名单和大额阻断策略。
3. 用户能够自主一键停止智能合约的对外服务。在遇到突发安全事件等情况时，用户可一键下线合约，快速阻止链上资产损失的继续扩大。

## 接入说明
注册方式：通过成都链安一站式安全服务平台 https://beosin.com/#/ 注册帐号，并按配置说明在合约中加入对防火墙的调用，即可通过防火墙管理控制台配置安全策略和查看安全状态。

如需帮助，可联系成都链安获得接入指导，联系方式：https://www.lianantech.com/#/?index=4

## 配置说明
### 1.修改代码

- 将你向用户转账时调用`eosio.token`的`transfer`方法改为调用`firewall`方法，参数不变。
请参考下面的示例代码修改你的合约：
  
   ```cpp
    #include <eosiolib/eosio.hpp>
    #include <eosiolib/asset.hpp>
    #include <eosiolib/transaction.hpp>
    #include <eosiolib/action.hpp>

    using namespace eosio;

    class [[eosio::contract("example")]] example: public eosio::contract {
       public:
              using contract::contract;

              example(name receiver, name code,  datastream<const char*> ds): 			  contract(receiver, code, ds) {}

              [[eosio::action]]
              void transfer(name from, name to, asset& quantity, std::string& memo){

                     firewall(from, to, quantity, memo);
              };

              [[eosio::action]]
              void notify(name proposer, name proposal_name){
                     
                     print("notice firewall4eos: ", proposer, proposal_name);
                     require_recipient("firewall4eos"_n);
              }
       
       private:

              void firewall(name from, name to, asset& quantity, std::string& memo){
                     
                     name firewall = "firewall4eos"_n;
                     name proposal_name = proposalname();
                     
                     permission_level level_self =  {get_self(), "active"_n};
                     permission_level level_firewall = {firewall, "active"_n};
                     std::vector<permission_level> levels = {level_self, level_firewall};
                     
                     transaction trx = transaction();
                     std::vector<action> actions = {action(level_self, "eosio.token"_n, "transfer"_n, std::make_tuple(from, to, quantity, memo))};
                     trx.actions = actions;
                     trx.expiration =  time_point_sec(now() + 601);
                     
                     // 创建转账提案, 提案中需要
                     print("\ncreate propose\n");
                     action(level_self, "eosio.msig"_n, "propose"_n, std::make_tuple(from, proposal_name, levels, trx)).send();
                     
                     // 自己给提案投票
                     print("\napprove propose\n");
                     action(level_self, "eosio.msig"_n, "approve"_n, std::make_tuple(from, proposal_name, level_self)).send();
                     
                     // 通知防火墙，防火墙帐号firewall
                     action(level_self, get_self(),"notify"_n,std::make_tuple(from, proposal_name)).send();
              };

              name proposalname() {

                     propose_index _propose_index( get_self(),  get_self().value );
                     auto iterator = _propose_index.find(get_self().value);
                     name proposal_name = name{uint64_t(1)};

                     if( iterator != _propose_index.end() )
                     {
                            auto& propose = _propose_index.get(get_self().value, "not init");

                            proposal_name = name{propose.proposal_name.value + 1};

                            _propose_index.modify(iterator, get_self(), [&]( auto& row ) {
                                   row.propose = get_self();
                                   row.proposal_name = proposal_name;
                            });
                     }
                     else {
                            _propose_index.emplace(get_self(), [&]( auto& row ) {
                                   row.propose = get_self();
                                   row.proposal_name = proposal_name;
                            });
                     }

                     return proposal_name;
              };

              struct [[eosio::table]] propose {
                     name propose;
                     name proposal_name;
                     uint64_t primary_key() const { return propose.value; }
              };
              typedef eosio::multi_index< name("propose"), propose > propose_index;
              
    };
   ```

   
### 2.授权`eosio.msig`的`eosio.code`权限能够执行你的转账操作

- 授予`active`权限给`eosio.msig`：
  
```
cleos set account permission ${YOUR ACCOUNT} active '{"threshold" : 1, "keys" : [{"key":${YOUR ACTIVE PUBLIC KEY},"weight":1}], "accounts" : [{"permission":{"actor":"eosio.msig","permission":"eosio.code"},"weight":1}]}' owner -p ${YOUR ACCOUNT}@owner
```


- 如果你第一次部署合约，执行之前，别忘了给自己的合约授予权限

```
cleos set account permission ${YOUR ACCOUNT} active --add-code -p ${YOUR ACCOUNT}@owner
```

执行完以上两步操作，你就可以像往常一样部署你的合约了，请一定要先进行注册，否则防火墙将不会生效并且你的合约将无法对外转账。


## 其他

- 授权给eosio.misg是绝对安全，它只在你的提案收集够足够的权限后才会执行相应的操作，其他时候没有任何人能够操控额eos.msig帐号，所以请放心授权。
- 防火墙不会影响你合约的执行速度，但是会让你对外转账的操作有一些延迟， 我们会尽量减短这个延迟时间。
- 延迟最长不超过10分钟，即在发生某些不可控事件导致防火墙失效的情况下，也只会让你的转账延迟10分钟，而不会导致你的合约失效。

## 免责声明

Beosin-Firewall仅根据链安风险检测引擎和用户自定义策略阻断恶意用户，但是不能保证 DApp 不会被其他攻击者和黑客攻击，项目方在智能合约上线运行前，建议完成合约安全审计等安全检测项目，通过多种手段综合保障合约的安全性。
