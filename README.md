# MviPattern
Android Mvi Pattern Example with Data Flow Reactive

## Component

### Repository
It is responsible for communicating with the true owner of the data (remote server, local database) and returning the domain object.

The Interface should not know who the owner of the Data is.

```kotlin
interface MessageRepository {
  suspend fun getMessages() : ImmutableList<Message>

  fun getMessageFlow() : Flow<ImmutableList<Message>>
}


class MessageRepositoryImpl @Inject constructor(
  // Remote Data Source
  private val messageApi : MessageApi,
  // Local Data Source
  private val messageDao : MessageDao
) : MessageRepository {

  override suspend fun getMessages() : ImmutableList<Message> {
    val messages = messageApi.getMessages().response
    messageDao.upsertMessages()
    return messages
  }

  override fun getMessageFlow() : Flow<ImmutableList<Message>> {
    return messageDao.getMessageFlow()
  }
}
```
### UseCase
Performs a single, defined domain function.  Receive injections from repositories and other UseCases.

```kotlin
class GetMessagesUseCase @Inject constructor(
  private val messsageRepository : MessageRepository
) {

   operator suspend fun invoke() : ImmutableList<Message> {
      return messageRepository.getMessages()
   }
}
```

### Middleware
Handle state changes for a single domain that integrates multiple features. Inject a UseCase.

From Middleware, this can be screen-dependent.

```kotlin

class MessageMiddleware @Inject constructor(
  private val getMessagesUseCase : GetMessagesUseCase
) : Middleware<MessagesState> {

  override suspend fun process(currentState : MessagesState, intent : Intent) : MessagesState {
      return when (intent) {
        is ScreenInitIntent -> currentState.reduce(messages = getMessagesUseCase())
        else -> currentState
      }
  }

}

```

### ViewModel
Control the overall behavior for one screen. Middleware is injected.

```kotlin

@HiltViewModel
class MessageListViewModel @Inject(
  private val messageMiddleware : MessageMiddleware
) : MviViewModel<MessageListState>() {

  override fun produceMiddlewares() : List<Middleware<in MessageListState>> {
    return listOf(messageMiddleware)
  }

  override fun produceInitState() : MessageListState {
    return MessageListState()
  }
}
```

## Flow Diagram
### User Interaction
![Image](/doc/diagram_userinteraction.png)
