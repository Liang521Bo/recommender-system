> This introduces a relatively basic recommendation algorithm, Collaborative Filtering. Based on the historical commodity recommendation purchased by the user, the item is collaboratively filtered.
>
> As more and more user information is collected, the recommendation system can draw a person's user portrait, and now more system user portraits are combined with on-site information to implement a recommendation system. In the next step I will implement a recommendation system based on user portraits.
>
## Recommender Algorithm
#### 1、Why Recommender system
RS will help users find the items they want, and on the other hand, expose the items to the user groups who are interested in them.

#### 2、How to evaluate Recommender system
##### Business Metrics
- Information Flow
    - Click rate：Number of clicks/Number of display. Provided certain number of display, the more clicks the better.
    - Average reading time:（1）Total reading time/Average number of clicks per person. The longer average reading time is, the more accurate the recommendation is.（2）Total usage time/Number of display. The longer total usage time the better.
- E-commerce
    - Conversion rate:（1）total transactions/number of display（2）turnover per unit time.
##### Recommendation coverage
Recommendation coverage = Unique recommended items/total number of items 
The higher recommendation coverage the better.
##### offline vs online
- offine: The user records is simulated through model and data statistics. 通过模型和数据，模拟用户记录，进行数据统计。
- online: ABTest，When Offine's algorithm index is not lower than the baseline, a part of the information flow can be used as a test. After running for a period of time, the difference between this information flow and the overall index will be counted to judge the quality of the new algorithm.
#### 3、Industrial application
1. Information flow: Toutiao、Baidu feed、UC News
2. E-commerce: Guess You might Like 
3. o2o，LBS:Recommender system based on geographic location information.
#### Architecture design
![](https://upload-images.jianshu.io/upload_images/6234504-a6d7ab5436e85cb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- App user terminal
- Web Api，Receive App user access information and call back-end RPC functions. WebApi tries not to do business logic, but completes work such as message queues and logging. Message queue: shaving peaks and filling valleys. Logging: Big data analysis, model training.
- Match：Personalized recall. Based on item or user recommendation rules, calculate the items that should be offered to the user.
- Rank:Recommend item sorting, model scoring, and determine the display order of items.
- Strategy:The recommendation system is based on the rules of business scenarios. Since the recall algorithm Match and the sorting algorithm Rank are both model-based, some scenarios can be customized to adjust the model results.
#### 4、Industrial System Architecture
![](https://upload-images.jianshu.io/upload_images/6234504-fa6de5d78c6fe4af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- model&KV：Offline model. Calculating recommendation results based on user behavior, the main functions (1) Calculate the similarity between items (2) Calculate users with similar behaviors (3) Calculate the ranking of a certain Label item, for example, if the user likes sports, calculate the ranking of all sports items . These results are written to KV storage.
- RecallServer&KV:The recall server reads the results from the KV server. Recall means to select the recommended service from the item collection. KV, key-value store (RDBMS, relational database). KV is more suitable for caching and has fast access speed.
- DocDetailServer:After obtaining the itemID from the KV store, query the item's detailed information through the Doc Detail Server, and send it to the user through template splicing.
- Deep learning model：User features like UserEmbeding, itemEmbeding are vectorized and stored in KV. Then K-nearest neighbor search algorithm is implemented.
## Item cf item-based collaborative filtering
#### a. principle
![](https://upload-images.jianshu.io/upload_images/6234504-53e48f8c4777c760.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

In this system, there are：
- User A B C D
- Item a b c d  
User A bought a、b、d, user B bought b、c、e。 
Create an index of items to users (**inverted index**)：item a corresponds to User AD; item b corresponds to user ADC. In the comparison of the results, the users who purchased items a and d have a greater degree of overlap, so a and d can be approximated for recommendation. That is, item a can be recommended to user C.
#### 2. formula
![Item CF公式](https://upload-images.jianshu.io/upload_images/6234504-2a1a7ff462654432.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- Sij，Represents the degree of acquaintance between item i and item j. Numerator: the intersection of users who have "acted" on item i and item j; denominator: the union of users who have "acted" on item i and item j.
- Puj，Calculate the rating of user U for item j, that is, user U has not acted on item j, and we want to predict this value. Scoring according to Sij (synergy matrix).
    - (1)Select the acting item N(u) for user U, and select k items in N(u) that have a score close to item j (target item), (the larger the score in the Sij matrix, the closer it is)   
    - (2)The weighted sum of the user U ratings (Rui) of these items is the user U's recommendation rating for item j
![惩罚热门商品](https://upload-images.jianshu.io/upload_images/6234504-d33ef4d882223998.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

In formula upgrade 1, the numerator is reducing, the penalty for inactive users is small, and the penalty for active users is large.
![时间惩罚](https://upload-images.jianshu.io/upload_images/6234504-7000c6effe665e9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

In the original formula, only one commodity is considered for user consumption, and it is not considered that users consume the same commodity in different periods. If the user consumes item i and item j, if the consumption time interval is closer, then the weight of this "co-occurrence" should be larger, and the further the interval is, the smaller the weight. Divide by the interval time on the numerator, penalizing the time interval effect.
## User CF User-based collaborative filtering recommendation algorithm
#### Principle
![user cf](https://upload-images.jianshu.io/upload_images/6234504-fd315b6658d34c0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
In this system, there are：
- User A B C D
- Item a b c d  

User A has consumed three items a, d, and b, and user D has consumed two items a and d. Through calculation, the items consumed by user A and user D have a high degree of acquaintance, then it is considered that user A and user D have similar interests, and the items purchased by user A can be recommended to user D. The above example can recommend item b to user D.
#### formula
![user cf](https://upload-images.jianshu.io/upload_images/6234504-544ba9e79948dc4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- Suv，The degree of acquaintance between user u and user v. The numerator is the intersection of user u and user v's acted items, and the denominator is the union of user u and user v's acted items. The more items that user u and user v have used at the same time, the greater the Suv, and the greater the user acquaintance.
- Pui, User u's predicted rating for item i
    - (1)Select the set of users U(i) who have acted on item i, and select k users in U(i) that are similar to user u (target user) (the larger the Suv, the more similar).
    - (2)Calculate the weighted score (Rvi).

![惩罚热门商品](https://upload-images.jianshu.io/upload_images/6234504-7b4f9ba1eb887da1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

If user u and user v have purchased many of the same popular items, it cannot indicate that the two users have acquaintance with interests and behaviors. 

In usercf formula upgrade 1, each item in the numerator is divided by the number of sales, then the contribution rate of each item ranges from 1 to 0. The more items are sold, the closer to 0, the smaller the contribution. This penalizes the influence of popular items on the calculation of the user collaboration matrix.
![惩罚购买间隔](https://upload-images.jianshu.io/upload_images/6234504-e6d7daa79bfb875f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

If user u and user v have purchased the same product at a longer time interval, the impact of the product on the user collaboration matrix should also be reduced.
## Comparison of ItemCF and UserCF
#### Real-time recommendation  
UserCF has poor real-time performance. Based on the user similarity matrix, users will not change their acquaintance with other users with a small amount of short-term behavior (the behavior does not change much, refer to Suv) during the process of using the system, so new items will not be recommended.  
ItemCF has high real-time performance. When a user acts on a new item, the related items of the new item will get a high rank according to the recommendation algorithm and are quickly recommended.
#### New Items, New User Recommendations
UserCf, cannot recommend to new users, since new users have no behavior. Cannot build user collaboration matrix. Cannot recommend new users based on similar users. New items are recommended by a user, and users similar to this user will get this new item recommendation.  
ItemCf, cannot recommend new items, because the item is not added to the synergy matrix. Simialr items to acting items can be recommended to new users. 
#### Interpretability of recommender systems
UserCf, based on similar user recommendations, it is difficult to describe the preferences of acquainted users. 
ItemCF, recommends based on the items the user has clicked on, with good interpretability.
#### ItemCf and UserCf application scenarios
- Performance: The calculation cost of constructing a similarity matrix is relatively high. The number of system users in the real environment is much larger than the number of products. ItemCF is used for performance considerations.
- Personalization: UserCf is suitable for scenarios that require items to be delivered in a timely manner and the personalization requirements are not strong. ItemCF is suitable for scenes with rich items and strong personalization.
- The application scenario of applying ItemCF can release new products in time in combination with other recall strategies, so ItemCF is more preferable.
