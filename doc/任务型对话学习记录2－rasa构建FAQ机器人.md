构建FAQ机器人

FAQ即 frequency asked questions，常见问题，通过收集常见问题，以对话交互的形式实现用户的问答．相当于检索类型的问答系统．可以通过rasa构建简单的FAQ问答机器人．

这部分参考官方文档，构建一个基础的FAQ项目．项目的构思是处理三种意图：打招呼，再见以及FAQ.


## １．新建一个新项目

执行命令

	rasa init
	
会生成完整项目的所有必备文件．包括

	actions.py
	config.yml
	credentials.yml
	data
		- nlu.md
		-stories.md
	domain.yml
	endpoints.yml
	models
	
在此基础之上，通过配置一些文件实现FAQ机器人的实现．

	- 定义全局变量
	- 准备训练数据
	- 配置语言理解和对话管理的方法策略
	- 训练并预测



##２．定义全局变量

我们的FAQ会识别的意图包括三种：greet,goodbye,faq.在domain文件中进行配置.对于faq，还需要配置一个内置的Retrieval Actions．对于FAQ而言，这个action必须使用respond_作为命名前缀．它的作用是将ResponseSelectors组件（后面会介绍）的输出发送给用户．除此之外，domain中还要配置greet和goodbye意图的回复消息．domain文件内容如下：

		intents:
		  - greet
		  - goodbye
		  - faq
		  
		actions:
		  - respond_faq
		  
	　　responses:
		  utter_greet:
		  - text: "Hey! How are you?"
		  utter_goodbye:
		  - text: "Bye"



## 3.准备训练数据
domain全局文件定义好以后，构建语言理解与对话管理训练数据集．
对于nlu意图识别训练数据，定义如下：

		## intent:greet
		- hey
		- hello
		- hi
		- good morning
		- good evening
		- hey there

		## intent:goodbye
		- bye
		- goodbye
		- see you around
		- see you later

		## intent: faq/ask_channels
		- What channels of communication does rasa support?
		- what channels do you support?
		- what chat channels does rasa uses
		- channels supported by Rasa
		- which messaging channels does rasa support?

		## intent: faq/ask_languages
		- what language does rasa support?
		- which language do you support?
		- which languages supports rasa
		- can I use rasa also for another laguage?
		- languages supported

分别对应三种意图其中 faq/ask_channels和 faq/ask_languages是faq意图下的两种问题，属于同一个意图，在上面这个例子中，nlu数据可以看成是一个标记的三分类文本数据训练集，rasa通过这个训练集来训练一个三分类的模型．


story数据用于训练对话管理模块．内容如下：

	## happy path
	* greet
	  - utter_greet

	## say goodbye
	* goodbye
	  - utter_goodbye

	## Some question from FAQ
	* faq
	 - respond_faq
	 
在这个简单的例子中，上面的数据可以理解为：当用户输入被识别为greet意图时候，就回复domain文件中定义的utter_greet内容．对于goodbye以此类推．对于faq意图，执行domain中定义actions:respond_faq.这个action会把ResponseSelectors组件的输出拿过来．

对于FAQ对话而言，还需要在data文件夹中增加一个responses.md文件，内容如下：

	## ask channels
	* faq/ask_channels
	  - We have a comprehensive list of [supported connectors](https://rasa.com/docs/core/connectors/), but if you don't see the one you're looking for, you can always create a custom connector by following

	## ask languages
	* faq/ask_languages
	  - You can use Rasa to build assistants in any language you want!

response.md文件中对f所有faq意图定义了相应的回复．

## 4. 定义config配置文件：

最后一步是在config文件中定义意图理解和对话管理使用的模型．整个文件的内容如下：

	language: en
	pipeline:
	  - name: WhitespaceTokenizer
	  - name: RegexFeaturizer
	  - name: LexicalSyntacticFeaturizer
	  - name: CountVectorsFeaturizer
	  - name: CountVectorsFeaturizer
	    analyzer: "char_wb"
	    min_ngram: 1
	    max_ngram: 4
	  - name: DIETClassifier
	    epochs: 100
	  - name: EntitySynonymMapper
	  - name: ResponseSelector
	    epochs: 100

	# Configuration for Rasa Core.
	# https://rasa.com/docs/rasa/core/policies/
	policies:
	  - name: MemoizationPolicy
	  - name: TEDPolicy
	    max_history: 5
	    epochs: 100
	  - name: MappingPolicy

其中language表示对话的语言，en是英文，zh表示中文．
pipeline定义了意图理解的模型．policies定义了对话管理的模型．
FAQ对话中用到的一个必备组建是ResponseSelectors．ResponseSelectors是一个监督式的检索模型，它的训练和预测过程如下，

1. 对用户的输入消息和定义的response进行特征提取．
2. 对它们分别进行特征学习．
3. 计算用户消息与response的相似度，对于正样例，最大化相似度，对于负样例，最小化相似度，学习模型并保存．
4. 预测时将用户的输入与所有的response进行比对，选择相似度最大的作为回复输出．

当用户的消息被识别为FAQ意图以后，就可以通过ResponseSelectors从预先定义的回复文本中检索生回复．生成的回复发送给respond_faq action返回给用户．


## 5.最后使用命令行工具进行训练和预测

训练：

	rasa train
	
预测：

	rasa shell

![ ](/home/wang/work/myproject/writing/dialogue/markdown/pic/1.png  "FAQ bot")


	



