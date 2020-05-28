# 功能组件
## 批量接口控制超时
```java
    public Map<Long, Map<String, Object>> execute(Map<Long, Map<String, Object>> inputParams, Long timeout, Map<String, Object> resultCache) {
        final long startTime = System.currentTimeMillis();
        final long endTime = startTime + timeout;
        Map<Long, Map<String, Object>> data = Maps.newHashMapWithExpectedSize(inputParams.size());
        CompletionService<Map<Long, Map<String, Object>>> completionService = new ExecutorCompletionService<>(executor);
        List<Future<Map<Long, Map<String, Object>>>> futureList = new ArrayList<>();
        inputParams.forEach((id, inputParam) -> {
            Future<Map<Long, Map<String, Object>>> submit = completionService.submit(new TaskExecutor(new Task(id, inputParam), varProcessContext, resultCache));
            futureList.add(submit);
        });

        try {
            for (Long id : inputParams.keySet()) {
                long timeLeft = endTime - System.currentTimeMillis();
                try {
                    Future<Map<Long, Map<String, Object>>> future = completionService.poll(timeLeft, TimeUnit.MILLISECONDS);
                    if (future == null) {
                        throw new TimeoutException("batch waiting timeout");
                    }
                    data.putAll(future.get());
                } catch (InterruptedException e) {
                    logger.error("BatchContainerExecutor InterruptedException", e);
                } catch (ExecutionException e) {
                    logger.error("BatchContainerExecutor ExecutionException", e);
                }
            }
        } catch (TimeoutException e) {
            for (Future<Map<Long, Map<String, Object>>> future : futureList) {
                future.cancel(true);
            }
            logger.error("BatchContainerExecutor出现超时异常,取消本组taskList剩余任务", e);
        }
        return data;
    }
```
