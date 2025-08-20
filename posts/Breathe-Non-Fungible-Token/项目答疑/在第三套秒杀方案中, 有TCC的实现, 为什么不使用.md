此处废弃了TCC的方案，主要是这个方案上会多次访问数据库，占用很多IO，导致CPU飙高的问题，所以改用非TCC方案

tradeApplicationService.newBuyPlusByTcc(orderCreateAndConfirmRequest);  

为了避免在创建订单的时候，confirm假失败（比如网络超时），导致库存不扣减的问题，这里需要查询最新的状态决定是否要发消息  
但是这里还是有可能出现因为网络延迟或者数据库异常而导致查询到的订单状态不是CONFIRM，但是后来又变成了CONFIRM的情况，所以需要做补偿，详见

NewBuyPlusMsgListener.newBuyPlusPreCancel  
SingleResponse<TradeOrderVO> response = orderFacadeService.getTradeOrder(orderCreateAndConfirmRequest.getOrderId());  

如果订单已经创建成功，则直接返回。不再需要做废单处理了。  

if (response.getSuccess() && response.getData() != null && response.getData().getOrderState() == TradeOrderState.CONFIRM) {  
     return LocalTransactionState.COMMIT_MESSAGE;  
}

	