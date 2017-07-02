# FirebaseRecyclerView(List DatabaseReference support)

## Following snippet could be used for senario like if you want to load recycler view from multiple firebase database urls or references or you want to create multiple card(child View) in single recycler view from multiple locations

### Implement following firebase recycler view for List of (firebase) database reference in single recycler View

Make FirebasebaseRecyclerAdapter.kt file as
```kotlin
abstract class FirebaseRecyclerAdapter<T, VH : RecyclerView.ViewHolder>
internal constructor(private val mModelClass: Class<T>,
                     protected var mModelLayout: Int,
                     protected var mViewHolderClass: Class<VH>,
                     private var mSnapshots: FirebaseArray)
    : RecyclerView.Adapter<VH>() {

    init {

        mSnapshots.setOnChangedListener(object : ChangeEventListener {
            override fun onChildChanged(type: ChangeEventListener.EventType, index: Int, oldIndex: Int) {
                this@FirebaseRecyclerAdapter.onChildChanged(type, index, oldIndex)
            }

            override fun onDataChanged() {
                this@FirebaseRecyclerAdapter.onDataChanged()
            }

            override fun onCancelled(error: DatabaseError) {
                this@FirebaseRecyclerAdapter.onCancelled(error)
            }
        })
    }

    /**
     * @param modelClass      Firebase will marshall the data at a location into
     * *                        an instance of a class that you provide
     * *
     * @param modelLayout     This is the layout used to represent a single item in the list.
     * *                        You will be responsible for populating an instance of the corresponding
     * *                        view with the data from an instance of modelClass.
     * *
     * @param viewHolderClass The class that hold references to all sub-views in an instance modelLayout.
     * *
     * @param ref             The Firebase location to watch for data changes. Can also be a slice of a location,
     * *                        using some combination of `limit()`, `startAt()`, and `endAt()`.
     */
    constructor(modelClass: Class<T>, modelLayout: Int,
                viewHolderClass: Class<VH>, ref: List<Query>)
            : this(modelClass, modelLayout, viewHolderClass, FirebaseArray(ref))


    fun cleanup() {
        mSnapshots.cleanup()
    }

    override fun getItemCount(): Int {
        return mSnapshots.count
    }

    fun getItem(position: Int): T? {
        return parseSnapshot(mSnapshots.getItem(position))
    }

    /**
     * This method parses the DataSnapshot into the requested type. You can override it in subclasses
     * to do custom parsing.

     * @param snapshot the DataSnapshot to extract the model from
     * *
     * @return the model extracted from the DataSnapshot
     */
    protected fun parseSnapshot(snapshot: DataSnapshot): T? {
        return snapshot.getValue(mModelClass)
    }

    fun getRef(position: Int): DatabaseReference {
        return mSnapshots.getItem(position).ref
    }

    fun getDataSnapshot(position: Int) : DataSnapshot{
        return mSnapshots.getItem(position)
    }

    override fun getItemId(position: Int): Long {
        // http://stackoverflow.com/questions/5100071/whats-the-purpose-of-item-ids-in-android-listview-adapter
        return mSnapshots.getItem(position).key.hashCode().toLong()
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val view = LayoutInflater.from(parent.context).inflate(viewType, parent, false)
        try {
            val constructor = mViewHolderClass.getConstructor(View::class.java)
            return constructor.newInstance(view)
        } catch (e: NoSuchMethodException) {
            throw RuntimeException(e)
        } catch (e: InvocationTargetException) {
            throw RuntimeException(e)
        } catch (e: InstantiationException) {
            throw RuntimeException(e)
        } catch (e: IllegalAccessException) {
            throw RuntimeException(e)
        }

    }

    override fun onBindViewHolder(viewHolder: VH, position: Int) {
        val model = getItem(position)
        if (model != null) {
            populateViewHolder(viewHolder, model, position)
        }
    }

    override fun getItemViewType(position: Int): Int {
        return mModelLayout
    }


    /**
     */
    open fun onChildChanged(type: ChangeEventListener.EventType, index: Int, oldIndex: Int) {
        when (type) {
            ChangeEventListener.EventType.ADDED -> notifyItemInserted(index)
            ChangeEventListener.EventType.CHANGED -> notifyItemChanged(index)
            ChangeEventListener.EventType.REMOVED -> notifyItemRemoved(index)
            ChangeEventListener.EventType.MOVED -> notifyItemMoved(oldIndex, index)
        }
    }

    /**
     */
    protected fun onDataChanged() {}

    /**
     */
    open fun onCancelled(error: DatabaseError) {
        Log.w(TAG, error.toException())
    }

    /**
     * Each time the data at the given Firebase location changes,
     * this method will be called for each item that needs to be displayed.
     * The first two arguments correspond to the mLayout and mModelClass given to the constructor of
     * this class. The third argument is the item's position in the list.
     *
     *
     * Your implementation should populate the view using the data contained in the model.

     * @param viewHolder The view to populate
     * *
     * @param model      The object containing the data used to populate the view
     * *
     * @param position   The position in the list of the view being populated
     */
    protected abstract fun populateViewHolder(viewHolder: VH, model: T, position: Int)

    fun notifyDataChanged(refs: List<Query>) {
        mSnapshots.cleanup()
        mSnapshots.setOnChangedListener(null)
        mSnapshots = FirebaseArray(refs)
        mSnapshots.setOnChangedListener(object : ChangeEventListener {
            override fun onChildChanged(type: ChangeEventListener.EventType, index: Int, oldIndex: Int) {
                this@FirebaseRecyclerAdapter.onChildChanged(type, index, oldIndex)
            }

            override fun onDataChanged() {
                this@FirebaseRecyclerAdapter.onDataChanged()
            }

            override fun onCancelled(error: DatabaseError) {
                this@FirebaseRecyclerAdapter.onCancelled(error)
            }
        })
        notifyDataSetChanged()
    }

    companion object {
        private val TAG = "CustomAdapter"
    }
}

internal class FirebaseArray(private val mQueries: List<Query>) : ChildEventListener, ValueEventListener {
    private var mListener: ChangeEventListener? = null
    private val mSnapshots = ArrayList<DataSnapshot>()

    init {
        for (mQuery in mQueries) {
            mQuery.addChildEventListener(this)
            mQuery.addValueEventListener(this)
        }
    }

    fun cleanup() {
        for (mQuery in mQueries) {
            mQuery.removeEventListener(this as ValueEventListener)
            mQuery.removeEventListener(this as ChildEventListener)
        }
    }

    val count: Int
        get() = mSnapshots.size

    fun getItem(index: Int): DataSnapshot {
        return mSnapshots[index]
    }

    private fun getIndexForKey(key: String): Int {
        var index = 0
        for (snapshot in mSnapshots) {
            if (snapshot.key == key) {
                return index
            } else {
                index++
            }
        }
        throw IllegalArgumentException("Key not found")
    }

    override fun onChildAdded(snapshot: DataSnapshot, previousChildKey: String?) {
        var index = 0
        if (previousChildKey != null) {
            index = getIndexForKey(previousChildKey) + 1
        }
        mSnapshots.add(index, snapshot)
        notifyChangedListeners(ChangeEventListener.EventType.ADDED, index)
    }

    override fun onChildChanged(snapshot: DataSnapshot, previousChildKey: String) {
        val index = getIndexForKey(snapshot.key)
        mSnapshots[index] = snapshot
        notifyChangedListeners(ChangeEventListener.EventType.CHANGED, index)
    }

    override fun onChildRemoved(snapshot: DataSnapshot) {
        val index = getIndexForKey(snapshot.key)
        mSnapshots.removeAt(index)
        notifyChangedListeners(ChangeEventListener.EventType.REMOVED, index)
    }

    override fun onChildMoved(snapshot: DataSnapshot, previousChildKey: String?) {
        val oldIndex = getIndexForKey(snapshot.key)
        mSnapshots.removeAt(oldIndex)
        var newIndex = mSnapshots.size
        if (previousChildKey != null) {
            newIndex = getIndexForKey(previousChildKey) + 1
        }
        mSnapshots.add(newIndex, snapshot)
        notifyChangedListeners(ChangeEventListener.EventType.MOVED, newIndex, oldIndex)
    }

    override fun onDataChange(dataSnapshot: DataSnapshot) {
        mListener?.onDataChanged()
    }

    override fun onCancelled(error: DatabaseError) {
        notifyCancelledListeners(error)
    }

    fun setOnChangedListener(listener: ChangeEventListener?) {
        mListener = listener
    }

    @JvmOverloads fun notifyChangedListeners(type: ChangeEventListener.EventType, index: Int, oldIndex: Int = -1) {
        if (mListener != null) {
            mListener?.onChildChanged(type, index, oldIndex)
        }
    }

    fun notifyCancelledListeners(error: DatabaseError) {
        if (mListener != null) {
            mListener?.onCancelled(error)
        }
    }
}
```

