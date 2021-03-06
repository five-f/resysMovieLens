# 推荐系统课程作业  
本次课程作业在`small-movielens`数据集的基础上，对用户、电影、评分数据进行了处理，然后根据`Pearson`相关系数计算出用户与其他用户之间的相似度，根据相似度进行推荐和预测评分，最后再根据数据集中的测试数据，计算该推荐系统的[`MAE`](https://baike.baidu.com/item/%E5%B9%B3%E5%9D%87%E7%BB%9D%E5%AF%B9%E8%AF%AF%E5%B7%AE/9383373?fr=aladdin)等。  
## **1. 数据描述**  
数据集来源于[MovieLens|GroupLens](https://grouplens.org/datasets/movielens/)网站。完整的数据集是由`943`个用户对`1682`个项目的`100000`个评分组成。每位用户至少对`20`部电影进行了评分。用户和项从`1`开始连续编号。数据是随机排列的。这是一个以选项卡分隔的：`用户id`|`项id`|`分级`|`时间戳`列表。从`1970年1月1日`UTC开始，时间戳是[`unix时间戳`](https://baike.baidu.com/item/unix%E6%97%B6%E9%97%B4%E6%88%B3/2078227?fr=aladdin)。在这些数据集的基础上，`small-movielens`还包括`u1-u5`，`ua`，`ub`等七组测试集`(*.base)`/训练集`(*.test)`数据。其中，`u1-u5`是以`8:2`的比例随机生成互斥的训练集和测试集；`ua`、`ub`是按照`9:1`的比例随机生成的互斥的训练集和测试集，不同的是，这两组数据的测试集中每个用户是对正好`10`部不同电影进行了评分。数据集如下图所示。  
![数据集](./pictures/dataSet.png "数据集")  
## **2. 变量说明**  
+ **users** :保存所有用户，不重复`list`类型  
存储结构：`[user1, user2,...]`  
+ **userWatchedMovie** :保存所有用户看过的所有电影，字典嵌套字典类型  
存储结构：`{user1:{movie1:rating1, movie2:rating2, ...}, user2:{...},...}`  
+ **movieUser** :保存用户与用户之间共同看过的电影，字典嵌套字典嵌套`list`  
存储结构：`{user1:{user2:[movie1,...],user3:[...],...},user2:{user3:[...],...},...}`  
+ **userSimilarity** :保存用户与用户之间的相似度（皮尔逊相似度）  
存储结构：`{user1:{user2:sim, user3:sim,...}, user2:{user3:sim, ...}, ...}`  
+ **allUserTopNSim** :保存每个用户都取前`n=10`个最相似的用户，以及相似度  
存储结构：`{user1:{user01:sim,user02:sim,...},user2:{user01:sim,...},...}`  
+ **recommendedMovies** :从最相似的用户中推荐，每个相似用户推荐两部，同时计算出预测值并保存在这个变量里  
存储结构：`{user1:{user01:{movie01:predictionRating,...},user02:[...],...},user2:{user01:[...],...},...}`  
+ **usersTest** :测试集文件中的所有用户  
存储结构：同`users`  
+ **userWatchedMovieTest** :测试集文件中所有用户看过的所有电影  
存储结构：同`userWatchedMovie`  
+ **movieAlsoInTest** :保存推荐的电影正好也在用户测试数据中看过的那一些电影，以便后面进行MAE计算  
存储结构：`{user1:[movie1,movie2,...],...}`  
+ **averageRating** :保存每个用户对被推荐的电影的预测平均分  
存储结构：`{user1:{movie01:[count,sumPreRating,averageRating],...},...}`  
+ **eachUserMAE** :保存对每个用户而言计算出的`MAE`  
存储结构：`{user1:MAE,user2:MAE,...}`  
## **3. 程序介绍**  
+ **首先对数据进行处理**，可以看到原始数据文件`u1.base`中的数据如下图所示  
![原始数据](./pictures/baseData.png "原始数据")  
数据是由`(userId, movieId, rating,timestamp)`四个部分组成，我们这里使用的是前三个数据属性。用变量`users`保存所有的用户`ID`,`userWatchedMovie`保存所有的用户看过的所有的电影  
![用户看过的电影](./pictures/userWatchedMovie.png "用户看过的电影")  
然后是对测试数据文件的读取，同上面做类似的处理  
+ **计算用户与用户之间共同看过的电影**  
![共同看过的电影](./pictures/movieUser.png "共同看过的电影")  
+ **计算用户与用户之间的相似度**  
在这个部分我们利用`Pearson`相关系数计算出两两用户之间的相似度  

            avgUserA = 0
            avgUserB = 0
            numerator = 0
            denominatorA = 0
            denominatorB = 0
            count = len(movieUser[a][b])
            factor = 0
            if count > 20:
                factor = 1.0
            else:
                if count < 0:
                    factor = 0
                else:
                    factor = (-0.0025 * count * count) + (0.1 * count)
            for movie in movieUser[a][b]:
                avgUserA += float(userWatchedMovie[a][movie])
                avgUserB += float(userWatchedMovie[b][movie])
            avgUserA = float(avgUserA / count)
            avgUserB = float(avgUserB / count)
            for m in movieUser[a][b]:
                tempA = float(userWatchedMovie[a][m]) - avgUserA
                tempB = float(userWatchedMovie[b][m]) - avgUserB
                numerator += tempA * tempB
                denominatorA += pow(tempA, 2) * 1.0
                denominatorB += pow(tempB, 2) * 1.0
            if denominatorA != 0 and denominatorB != 0:
                userSimilarity[a][b] = factor * (numerator / (sqrt(denominatorA * denominatorB)))
            else:
                userSimilarity[a][b] = 0

+ **每个用户都取前`n`个最相似的用户，以便后续进行推荐，例如`n=10`**  

        for compareUserId in users:
            if currentUserId == compareUserId:
                break
            else:
                singleUserSim[compareUserId] = userSimilarity[compareUserId][currentUserId]
        if int(currentUserId) != len(users):
            singleUserSim.update(userSimilarity[currentUserId])
        singleSortedSim = sorted(singleUserSim.items(), key=lambda item: item[1], reverse=True)
        singleTopN = singleSortedSim[:n]
        for single in singleTopN:
            allUserTopNSim[currentUserId][single[0]] = single[1]

    ![TopN个相似用户](./pictures/allUserTop10Sim.png "TopN个相似用户")  
+ **从最相似的用户中推荐number部电影**，例如每个相似用户推荐两部，那么每个用户就能得到推荐的`20`部电影  

		if movie not in userWatchedMovie[oneUser].keys():
			if int(oneUser) < int(simUser):
				length = len(movieUser[oneUser][simUser])
				sumOne = 0.0
				sumSim = 0.0
				for i in movieUser[oneUser][simUser]:
					sumOne += userWatchedMovie[oneUser].get(i)
					sumSim += userWatchedMovie[simUser].get(i)
				sumSim += userWatchedMovie[simUser].get(movie)
				avgOneUser = sumOne / length
				avgSimUser = sumSim / (length + 1)
				predictionRating = avgOneUser + (userWatchedMovie[simUser][movie] - avgSimUser)
				recommendedMovies[oneUser][simUser][movie] = predictionRating
				number += 1

    ![推荐和预测评分](./pictures/recoMovieWithRating.png "推荐和预测评分")  
+ **从推荐的电影和测试集中找到一起看过的电影**  

        movieAlsoInTest = {}
        for oneUser in usersTest:
			movieAlsoInTest.setdefault(oneUser, [])
			for simUser in recommendedWithRating[oneUser].keys():
				for movie in recommendedWithRating[oneUser][simUser].keys():
					if movie in userWatchedMovieTest[oneUser].keys():
						movieAlsoInTest[oneUser].append(movie)
					else:
						continue

    ![movieAlsoInTest](./pictures/movieAlsoInTest02.png "测试集中用户也看过的电影")  
+ **计算每个用户被推荐的每部电影的次数和平均分**  

		averageRating = {}
		for oneUser in usersTest:
			averageRating.setdefault(oneUser, {})
			for simUser in recommendedWithRating[oneUser].keys():
				for movie in recommendedWithRating[oneUser][simUser].keys():
					averageRating[oneUser].setdefault(movie, [0, 0.0, 0.0])
					averageRating[oneUser][movie][0] += 1
					averageRating[oneUser][movie][1] += recommendedWithRating[oneUser][simUser].get(movie)
			for each in averageRating[oneUser].keys():
				verageRating[oneUser][each][2] = averageRating[oneUser][each][1] / averageRating[oneUser][each][0]

    ![推荐平均分](./pictures/averageRating01.png "推荐平均分")  
+ **计算`MAE`**  

		eachUserMAE = {}
		for oneUser in averageRating.keys():
			count = 0
			sumD = 0.0
			eachUserMAE.setdefault(oneUser, 0.0)
			for movie in movieAlsoInTest[oneUser]:
				count += 1
				sumD += abs(averageRating[oneUser][movie][2] - userWatchedMovieTest[oneUser].get(movie))
			if count == 0:
				eachUserMAE[oneUser] = -1
			else:
				eachUserMAE[oneUser] = sumD / count

    ![用户MAE](./pictures/eachUserMAE01.png "每个用户的MAE")  

		recommendedMAE = 0.0
		countMAE = 0
		for oneUser in eachUserMAE.keys():
			if eachUserMAE[oneUser] != -1:
				countMAE += 1
				recommendedMAE += eachUserMAE[oneUser]
		recommendedMAE = recommendedMAE / countMAE

    ![系统MAE](./pictures/recsysMAE.png "推荐系统平均MAE")  
    另外可以引入**预测准确度**或者说**命中率**  
+ **实验结果**  
笔者对`u1-u5`以及`ua-ub`数据集中的数据分别进行了实验测试，并统计出如下结果  
![表格分析01](./pictures/table01.png "表格分析")  
从表格中可以看出，不同的因素会对推荐系统的`MAE`造成不同程度的影响，考虑到图片表格中的数据还并不是很直观，所以我们把上图表格中的部分数据抽取出来进行更为直观的比较。笔者主要从两个变量上考虑了对系统MAE的影响：**最相似用户数量TOPN**和**推荐的电影数AMOUNT**。笔者在不同的测试集下针对这两个变量做了相关的实验，结果如下表以及下图所示。  

    AMOUNT|u1|u4|ub
    :--:|:--:|:--:|:--:
    1|0.875733|0.880698|0.957882
    2|0.822871|0.818759|0.901364
    3|0.795623|0.825441|0.89594
    4|0.782709|0.828352|0.863475
    5|0.806841|0.803826|0.856949
    6|0.815769|0.813643|0.839806
    7|0.799495|0.811279|0.825822
    8|0.808569|0.815092|0.793608

    ![图片分析01](./pictures/chart1.png "不同AMOUNT下MAE比较")  
    在这里，最相似用户数是不变的，即`TOPN=10`  
    接下来，让`AMOUNT=5`作为固定值，比较了不同`TOPN`对系统MAE的影响，结果如下所示。  
    
    TOPN|u1|u4
    :--:|:--:|:--:
    5|0.83006|0.866108
    10|0.806841|0.803826
    15|0.805935|0.800955
    20|0.795955|0.797959
    25|0.794347|0.795494  
    
    ![图片分析02](./pictures/chart2.png "不同TOPN下MAE比较")  

一些考虑：  
在给用户推荐的电影数AMOUT不变的情况下，TOPN越大，好像MAE会越小，甚至会比0.75要低，当然到一定程度大小后（>50），会出现错误。有如下猜测：  
**[A]** 推荐算法本身的性能问题  
**[B]** 相似性计算的其他约束性条件未考虑  
**[C]** 数据集过小，电影的总数量和用户总数量过少(笔者尝试将更大数据集（>100M）分成训练集和测试集，遗憾的是没有成功，对python数据操作还不熟悉)  
**[D]** 用户看过的电影数量过少等  

@Date : 2019/4/1  
@Author : [Freator Tang](https://github.com/freator)  
@Email : bingcongtang@gmail.com
