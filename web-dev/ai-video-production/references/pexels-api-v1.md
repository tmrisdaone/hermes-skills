# Pexels API V1 Implementation Notes

## Response Structure
The `pexels-api` Python wrapper `.search()` method returns data that may vary between dictionary-style and list-style responses.

### Recommended Parsing Pattern
To avoid `KeyError: 'videos'`, use a flexible extraction pattern:

```python
search_results = self.pexels.search(keyword, page=1, results_per_page=5)

videos = []
if isinstance(search_results, dict) and 'videos' in search_results:
    videos = search_results['videos']
elif isinstance(search_results, list):
    videos = search_results

if not videos:
    return None

video_data = random.choice(videos)
video_url = None
if isinstance(video_data, dict):
    # The API can return 'video_files' (list) or a direct 'url'
    if 'video_files' in video_data:
        video_url = video_data['video_files'][0]['link']
    elif 'url' in video_data:
        video_url = video_data['url']
```

## Pitfalls
- **Method Name**: Use `.search()`. Do not use `.search_videos()` as it is not present in the `pexels-api` wrapper.
- **Empty Results**: Always check if the result list is empty before calling `random.choice()` to prevent `IndexError`.
