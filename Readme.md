# Deep RL Trader (Duel DQN) Implemented using Keras-RL

This repo contains 
1. Trading environment(OpenAI Gym) for trading crypto currency  
2. Duel Deep Q Network  
Agent is implemented using `keras-rl`(https://github.com/keras-rl/keras-rl)     
  
Agent is expected to learn useful action sequences to maximize profit in a given environment.  
Environment limits agent to either buy, sell, hold stock(coin) at each step.  
If an agent decides to take a 
⋅⋅* LONG position it will initiate sequence of action such as `buy- hold- hold- sell`    
⋅⋅* for a SHORT position vice versa (e.g.) `sell - hold -hold -buy`.    

Only a single position can be opened per trade. 
⋅⋅* Thus invalid action sequence like `buy - buy` will be considered `buy- hold`.   

Reward is given
⋅⋅* when the position closes or
⋅⋅* an episode is finished.   
  
This type of sparse reward granting scheme takes longer to train but is most successful at learning long term dependencies.  

Agent decides optimal action by observing its environment. 
⋅⋅* Trading environment will emit features derived from ohlcv-candles(the window size can be configured). 
⋅⋅* Thus, input given to the agent is of the shape `(window_size, n_features)`.  

With some modification it can easily be applied to stocks, futures or foregin exchange as well.

### Prerequisites

keras-rl, numpy, tensorflow ... etc

```python
pip install -r requirements.txt
```

## Getting Started 

```python
# create environment
# OPTIONS
ENV_NAME = 'OHLCV-v0'
TIME_STEP = 30
PATH_TRAIN = "./data/train/"
PATH_TEST = "./data/test/"
env = OhlcvEnv(TIME_STEP, path=PATH_TRAIN)
env_test = OhlcvEnv(TIME_STEP, path=PATH_TEST)

# random seed
np.random.seed(123)
env.seed(123)

# create_model
nb_actions = env.action_space.n
model = create_model(shape=env.shape, nb_actions=nb_actions)
print(model.summary())


# create memory
memory = SequentialMemory(limit=50000, window_length=TIME_STEP)

# create policy
policy = EpsGreedyQPolicy()# policy = BoltzmannQPolicy()

# create agent
# you can specify the dueling_type to one of {'avg','max','naive'}
dqn = DQNAgent(model=model, nb_actions=nb_actions, memory=memory, nb_steps_warmup=200,
               enable_dueling_network=True, dueling_type='avg', target_model_update=1e-2, policy=policy,
               processor=NormalizerProcessor())
dqn.compile(Adam(lr=1e-3), metrics=['mae'])


```python
# now train and test agent
while True:
    # train
    dqn.fit(env, nb_steps=5500, nb_max_episode_steps=10000, visualize=False, verbose=2)
    try:
        # validate
        info = dqn.test(env_test, nb_episodes=1, visualize=False)
        n_long, n_short, total_reward, portfolio = info['n_trades']['long'], info['n_trades']['short'], info[
            'total_reward'], int(info['portfolio'])
        np.array([info]).dump(
            './info/duel_dqn_{0}_weights_{1}LS_{2}_{3}_{4}.info'.format(ENV_NAME, portfolio, n_long, n_short,
                                                                        total_reward))
        dqn.save_weights(
            './model/duel_dqn_{0}_weights_{1}LS_{2}_{3}_{4}.h5f'.format(ENV_NAME, portfolio, n_long, n_short,
                                                                        total_reward),
            overwrite=True)
    except KeyboardInterrupt:
        continue

```

## Configuring Agent
```python
## simply plug in any keras model :)
def create_model(shape, nb_actions):
    model = Sequential()
    model.add(CuDNNLSTM(64, input_shape=shape, return_sequences=True))
    model.add(CuDNNLSTM(64))
    model.add(Dense(32))
    model.add(Activation('relu'))
    model.add(Dense(nb_actions, activation='linear'))
```

## Running 
[Verbose] While training or testing, environment will print out (current_tick , # Long, # Short, Portfolio)  
[Portfolio] initial portfolio starts with 100*10000(krw-won),   
reflects change in portfolio value if the agent had invested 100% of its balance every time it opened a position.  
[Reward] simply pct earning per trade.  

## Inital Result

![trade](https://github.com/miroblog/limit_orderbook_prediction/blob/master/image/features.png)  
![partial_trade](https://github.com/miroblog/limit_orderbook_prediction/blob/master/image/features.png)
![cum_return](https://github.com/miroblog/limit_orderbook_prediction/blob/master/image/features.png)

total cumulative return :  3.670099054203348
portfolio value 1000000 -> [29415305.46593453]

## Authors

* **Lee Hankyol** - *Initial work* - [deep_rl_trader](https://github.com/miroblog/deep_rl_trader)

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
