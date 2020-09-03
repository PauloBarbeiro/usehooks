---
templateKey: post
title: useFirestoreQuery
ogImage: og-use-firestore-query.jpg
date: "2020-08-11"
gist: https://gist.github.com/gragland/8f9495deda2d7c171fcd1b4ecddf2ebb
composes: ["useMemoCompare"]
links:
  - url: https://github.com/tannerlinsley/react-query
    name: React Query
    description: Data fetching library that has a similar useQuery hook and inspired the API for this example.
  - url: https://github.com/nandorojo/swr-firestore
    name: SWR Firestore
    description: Firestore query hooks built on top of SWR
code: "\/\/ Usage\r\nfunction ProfilePage({ uid }) {\r\n  \/\/ Subscribe to Firestore document\r\n  const { data, status, error } = useFirestoreQuery(\r\n    firestore.collection(\"profiles\").doc(uid)\r\n  );\r\n  \r\n  if (status === \"loading\"){\r\n    return \"Loading...\"; \r\n  }\r\n  \r\n  if (status === \"error\"){\r\n    return `Error: ${error.message}`;\r\n  }\r\n\r\n  return (\r\n    <div>\r\n      <ProfileHeader avatar={data.avatar} name={data.name} \/>\r\n      <Posts posts={data.posts} \/>\r\n    <\/div>\r\n  );\r\n}\r\n\r\n\/\/ Reducer for hook state and actions\r\nconst reducer = (state, action) => {\r\n  switch (action.type) {\r\n    case \"idle\":\r\n      return { status: \"idle\", data: undefined, error: undefined };\r\n    case \"loading\":\r\n      return { status: \"loading\", data: undefined, error: undefined };\r\n    case \"success\":\r\n      return { status: \"success\", data: action.payload, error: undefined };\r\n    case \"error\":\r\n      return { status: \"error\", data: undefined, error: action.payload };\r\n    default:\r\n      throw new Error(\"invalid action\");\r\n  }\r\n}\r\n\r\n\/\/ Hook\r\nfunction useFirestoreQuery(query) {\r\n  \/\/ Our initial state\r\n  \/\/ Start with an \"idle\" status if query is falsy, as that means hook consumer is\r\n  \/\/ waiting on required data before creating the query object.\r\n  \/\/ Example: useFirestoreQuery(uid && firestore.collection(\"profiles\").doc(uid))\r\n  const initialState = { \r\n    status: query ? \"loading\" : \"idle\", \r\n    data: undefined, \r\n    error: undefined \r\n  };\r\n  \r\n  \/\/ Setup our state and actions\r\n  const [state, dispatch] = useReducer(reducer, initialState);\r\n  \r\n  \/\/ Get cached Firestore query object with useMemoCompare (https:\/\/usehooks.com\/useMemoCompare)\r\n  \/\/ Needed because firestore.collection(\"profiles\").doc(uid) will always being a new object reference\r\n  \/\/ causing effect to run -> state change -> rerender -> effect runs -> etc ...\r\n  \/\/ This is nicer than requiring hook consumer to always memoize query with useMemo.\r\n  const queryCached = useMemoCompare(query, prevQuery => {\r\n    \/\/ Use built-in Firestore isEqual method to determine if \"equal\"\r\n    return prevQuery && query && query.isEqual(prevQuery);\r\n  });\r\n\r\n  useEffect(() => {\r\n    \/\/ Return early if query is falsy and reset to \"idle\" status in case\r\n    \/\/ we're coming from \"success\" or \"error\" status due to query change.\r\n    if (!queryCached) {\r\n      dispatch({ type: \"idle\" });\r\n      return;\r\n    }\r\n    \r\n    dispatch({ type: \"loading\" });\r\n    \r\n    \/\/ Subscribe to query with onSnapshot\r\n    \/\/ Will unsubscribe on cleanup since this returns an unsubscribe function\r\n    return queryCached.onSnapshot(\r\n      response => {\r\n        \/\/ Get data for collection or doc\r\n        const data = response.docs\r\n          ? getCollectionData(response)\r\n          : getDocData(response);\r\n        \r\n        dispatch({ type: \"success\", payload: data });\r\n      },\r\n      error => {\r\n        dispatch({ type: \"error\", payload: error });\r\n      }\r\n    );\r\n    \r\n  }, [queryCached]); \/\/ Only run effect if queryCached changes\r\n\r\n  return state;\r\n}\r\n\r\n\/\/ Get doc data and merge doc.id\r\nfunction getDocData(doc) {\r\n  return doc.exists === true ? { id: doc.id, ...doc.data() } : null;\r\n}\r\n\r\n\/\/ Get array of doc data from collection\r\nfunction getCollectionData(collection) {\r\n  return collection.docs.map(getDocData);\r\n}"
---

This hook makes it super easy to subscribe to data in your Firestore database without having to worry about state management. Instead of calling Firestore's `query.onSnapshot()` method you simply pass a query to `useFirestoreQuery()` and you get back everything you need, including `status`, `data`, and `error`. Your component will re-render when data changes and your subscription will be automatically removed when the component unmounts. Our example even supports dependent queries where you can wait on needed data by passing a falsy value to the hook. Read through the recipe and comments below to see how it works.