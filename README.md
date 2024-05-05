#初始化模块
def initialize(context):
    #将上证50（000016.SS）设置为参考基准,否则默认是000300.SS
    set_benchmark('000016.SS')
    #设置佣金费率，默认是万2佣金费率和5元的最低交易硼金
    set_commission(commission_ratio=0.0002, min_commission=5.0)
    #专没置据定滑点，即委托价格与展后的成交价格的价差，默认是0
    set_fixed_slippage(fixedslippage = 0.2)
    #设置滑点比例，数认为0.1,具体影响还与当时成交数量与该周期最大可成交量有关
    set_slippage(slippage = 0.2)
    #没置成交比例，默认0.25,即搭本周期最大成交数量为本周期市场可成交意量的四分之一
    set_volume_ratio(volume_ratio = 0.25)
    #设置交易股票池
    g.security='600520.SS'
    set_universe(g.security)

#盘前处理
def before_trading_start(context,data):

    #获取当日的股票池
    g.stock_list =get_index_stocks('000016.SS')
    
    #单获取服票的状态ST、停牌、退市
    st_status = get_stock_status(g.stock_list,'ST')
    halt_status = get_stock_status(g.stock_list,'HALT')
    delisting_status = get_stock_status(g.stock_list,'DELISTING')
    
    #将三种状态的股票剔除当日的股票池
    for stock in g.stock_list.copy():
        if st_status[stock] or halt_status[stock] or delisting_status[stock]:
            g.stock_list.remove(stock)
   #获取股票历史数据
    history = get_history(10,'1d',['close','volume'],g.stock_list, fq='pre')
    history = history.swapaxes("minor_axis","items")
    g.close_df_dict ={}
    for stock in g.stock_list.copy():
         #log.info(stock)
         df = history[stock]
         #过滤历史停牌的数妮
         df = df[df['volume']>0]
         #如果非停碑日期>=10,#增加到历史数氮存储字典，否则移除
         if len(df.index)<10:
             g.stock_list.remove(stock)
             continue
             close_array = df['close'].values[:]
             g.close_df_dict[stock] = close_array
         #设置今天的股票池
         set_universe(g.stock_list)

#盘中运行
def handle_data(context,data):
    
    #获取全市场股票，选最近10个交易日K线
    g.stock_list = get_Ashares()
    history= get_history(10, frequency='1d', field=['open', 'close', 'high'], security_list=g.stock_list, fq='pre', include=False, is_dict=True)
    
    #遍历股票列表
    for stock in g.stock_list:  
        # 从history字典中获取当前股票的历史数据
        bar_history = history[stock]
        #取得之前的收盘价
        if len(bar_history) >= 5:  # 确保有足够的历史数据 
           Threedaysago_close = bar_history['close'][-4]  
           Fourdaysago_close = bar_history['close'][-5]  
           Twodaysago_close = bar_history['close'][-3]  
           yesterday_close = bar_history['close'][-2]  
           price_ = bar_history['price'][-1]  
           high_=bar_history['high']
           # 买入条件  
           condition1 = Threedaysago_close / Fourdaysago_close < 1.08  
           condition2 = Twodaysago_close / Threedaysago_close > 1.092  
           condition3 = yesterday_close / Twodaysago_close < 1.08  
           condition4 = price / yesterday_close > 1.08  
              
           if condition1 and condition2 and condition3 and condition4:  
              # 买入股票  
              order_market(stock, 100,0)  # 注意这里使用stock代替security  
              log.info('buy %s' % stock)  
        else:  
           log.info('Insufficient history data for %s' % stock)  
        # 卖出条件（这里只是一个示例，您可能需要更复杂的逻辑） 
        if context.portfolio.positions[stock].quantity > 0:  # 确保持有该股票  
           if price_ / yesterday_close > 1.06 and price_/high<0.99:  
             # 卖出股票  
             order_target(stock, 0) 
           elif price_ / yesterday_close < 0.97: 
               order_target(stock, 0)            
               log.info('sell %s' % stock)  
#盘后处理
def after_trading_end(context,data):
    position_list =[]
    for stock in context.portfolio.positions:
        if context.portfolio.positions[stock].amount !=0:
            position_list.append(stock)
