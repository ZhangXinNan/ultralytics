---
comments: true
description: Explore and analyze CV datasets with Ultralytics Explorer API, offering SQL, vector similarity, and semantic searches for efficient dataset insights.
keywords: Ultralytics Explorer API, Dataset Exploration, SQL Queries, Vector Similarity Search, Semantic Search, Embeddings Table, Image Similarity, Python API for Datasets, CV Dataset Analysis, LanceDB Integration
---

# Ultralytics Explorer API

## Introduction

The Explorer API is a Python API for exploring your datasets. It supports filtering and searching your dataset using SQL queries, vector similarity search and semantic search.

## Installation

Explorer depends on external libraries for some of its functionality. These are automatically installed on usage. To manually install these dependencies, use the following command:

```bash
pip install ultralytics[explorer]
```

## Usage

```python
from ultralytics import Explorer

# Create an Explorer object
explorer = Explorer(data='coco128.yaml', model='yolov8n.pt')

# Create embeddings for your dataset
explorer.create_embeddings_table()

# Search for similar images to a given image/images
dataframe = explorer.get_similar(img='path/to/image.jpg')

# Or search for similar images to a given index/indices
dataframe = explorer.get_similar()(idx=0)
```

## 1. Similarity Search

Similarity search is a technique for finding similar images to a given image. It is based on the idea that similar images will have similar embeddings. Once the embeddings table is built, you can get run semantic search in any of the following ways:

- On a given index or list of indices in the dataset: `exp.get_similar(idx=[1,10], limit=10)`
- On any image or list of images not in the dataset: `exp.get_similar(img=["path/to/img1", "path/to/img2"], limit=10)`

In case of multiple inputs, the aggregate of their embeddings is used.

You get a pandas dataframe with the `limit` number of most similar data points to the input, along with their distance in the embedding space. You can use this dataset to perform further filtering

!!! Example "Semantic Search"

    === "Using Images"

        ```python
        from ultralytics import Explorer

        # create an Explorer object
        exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
        exp.create_embeddings_table()

        similar = exp.get_similar(img='https://ultralytics.com/images/bus.jpg', limit=10)
        print(similar.head())

        # Search using multiple indices
        similar = exp.get_similar(
                                img=['https://ultralytics.com/images/bus.jpg',
                                     'https://ultralytics.com/images/bus.jpg'],
                                limit=10
                                )
        print(similar.head())
        ```

    === "Using Dataset Indices"

        ```python
        from ultralytics import Explorer

        # create an Explorer object
        exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
        exp.create_embeddings_table()

        similar = exp.get_similar(idx=1, limit=10)
        print(similar.head())

        # Search using multiple indices
        similar = exp.get_similar(idx=[1,10], limit=10)
        print(similar.head())
        ```

### Plotting Similar Images

You can also plot the similar images using the `plot_similar` method. This method takes the same arguments as `get_similar` and plots the similar images in a grid.

!!! Example "Plotting Similar Images"

    === "Using Images"

        ```python
        from ultralytics import Explorer

        # create an Explorer object
        exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
        exp.create_embeddings_table()

        plt = exp.plot_similar(img='https://ultralytics.com/images/bus.jpg', limit=10)
        plt.show()
        ```

    === "Using Dataset Indices"

        ```python
        from ultralytics import Explorer

        # create an Explorer object
        exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
        exp.create_embeddings_table()

        plt = exp.plot_similar(idx=1, limit=10)
        plt.show()
        ```

## 2. SQL Querying

You can run SQL queries on your dataset using the `sql_query` method. This method takes a SQL query as input and returns a pandas dataframe with the results.

!!! Example "SQL Query"

    ```python
    from ultralytics import Explorer

    # create an Explorer object
    exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
    exp.create_embeddings_table()

    df = exp.sql_query("WHERE labels LIKE '%person%' AND labels LIKE '%dog%'")
    print(df.head())
    ```

### Plotting SQL Query Results

You can also plot the results of a SQL query using the `plot_sql_query` method. This method takes the same arguments as `sql_query` and plots the results in a grid.

!!! Example "Plotting SQL Query Results"

    ```python
    from ultralytics import Explorer

    # create an Explorer object
    exp = Explorer(data='coco128.yaml', model='yolov8n.pt')
    exp.create_embeddings_table()

    df = exp.sql_query("WHERE labels LIKE '%person%' AND labels LIKE '%dog%'")
    print(df.head())
    ```

## 3. Working with embeddings Table (Advanced)

You can also work with the embeddings table directly. Once the embeddings table is created, you can access it using the `Explorer.table`

!!! Tip "Explorer works on [LanceDB](https://lancedb.github.io/lancedb/) tables internally. You can access this table directly, using `Explorer.table` object and run raw queries, push down pre- and post-filters, etc."

    ```python
    from ultralytics import Explorer

    exp = Explorer()
    exp.create_embeddings_table()
    table = exp.table
    ```

Here are some examples of what you can do with the table:

### Get raw Embeddings

!!! Example

    ```python
    from ultralytics import Explorer

    exp = Explorer()
    exp.create_embeddings_table()
    table = exp.table

    embeddings = table.to_pandas()["vector"]
    print(embeddings)
    ```

### Advanced Querying with pre and post filters

!!! Example

    ```python
    from ultralytics import Explorer

    exp = Explorer(model="yolov8n.pt")
    exp.create_embeddings_table()
    table = exp.table

    # Dummy embedding
    embedding = [i for i in range(256)]
    rs = table.search(embedding).metric("cosine").where("").limit(10)
    ```

### Create Vector Index

When using large datasets, you can also create a dedicated vector index for faster querying. This is done using the `create_index` method on LanceDB table.

```python
    table.create_index(num_partitions=..., num_sub_vectors=...)
```

Find more details on the type vector indices available and parameters [here](https://lancedb.github.io/lancedb/ann_indexes/#types-of-index) In the future, we will add support for creating vector indices directly from Explorer API.

## 4. Embeddings Applications

You can use the embeddings table to perform a variety of exploratory analysis. Here are some examples:

### Similarity Index

Explorer comes with a `similarity_index` operation:

- It tries to estimate how similar each data point is with the rest of the dataset.
- It does that by counting how many image embeddings lie closer than `max_dist` to the current image in the generated embedding space, considering `top_k` similar images at a time.

It returns a pandas dataframe with the following columns:

- `idx`: Index of the image in the dataset
- `im_file`: Path to the image file
- `count`: Number of images in the dataset that are closer than `max_dist` to the current image
- `sim_im_files`: List of paths to the `count` similar images

!!! Tip

    For a given dataset, model, `max_dist` & `top_k` the similarity index once generated will be reused. In case, your dataset has changed, or you simply need to regenerate the similarity index, you can pass `force=True`.

!!! Example "Similarity Index"

    ```python
    from ultralytics import Explorer

    exp = Explorer()
    exp.create_embeddings_table()

    sim_idx = exp.similarity_index()
    ```

You can use similarity index to build custom conditions to filter out the dataset. For example, you can filter out images that are not similar to any other image in the dataset using the following code:

```python
import numpy as np

sim_count = np.array(sim_idx["count"])
sim_idx['im_file'][sim_count > 30]
```

### Visualize Embedding Space

You can also visualize the embedding space using the plotting tool of your choice. For example here is a simple example using matplotlib:

```python
import numpy as np
from sklearn.decomposition import PCA
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# Reduce dimensions using PCA to 3 components for visualization in 3D
pca = PCA(n_components=3)
reduced_data = pca.fit_transform(embeddings)

# Create a 3D scatter plot using Matplotlib Axes3D
fig = plt.figure(figsize=(8, 6))
ax = fig.add_subplot(111, projection='3d')

# Scatter plot
ax.scatter(reduced_data[:, 0], reduced_data[:, 1], reduced_data[:, 2], alpha=0.5)
ax.set_title('3D Scatter Plot of Reduced 256-Dimensional Data (PCA)')
ax.set_xlabel('Component 1')
ax.set_ylabel('Component 2')
ax.set_zlabel('Component 3')

plt.show()
```

Start creating your own CV dataset exploration reports using the Explorer API. For inspiration, check out the

# Apps Built Using Ultralytics Explorer

Try our GUI Demo based on Explorer API

# Coming Soon

- [ ] Merge specific labels from datasets. Example - Import all `person` labels from COCO and `car` labels from Cityscapes
- [ ] Remove images that have a higher similarity index than the given threshold
- [ ] Automatically persist new datasets after merging/removing entries
- [ ] Advanced Dataset Visualizations