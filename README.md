Java provides a neat way to carry out long lasting jobs without have to worry about the complexity of threads or lack of responsiveness in an application (by application we mean Swing applications).  It is called <code>SwingWorker</code> (<a href="http://java.sun.com/javase/7/docs/api/javax/swing/SwingWorker.html" target="_blank">Java Doc</a>).  It is not the latest thing on Earth (released with Java 1.6) and you may have already read about it.  In this article we will see how to use it and why it was created to start with.


All code listed below is available at: <a href="https://github.com/javacreed/swing-worker-example" target="_blank">https://github.com/javacreed/swing-worker-example</a>.  Most of the examples will not contain the whole code and may omit fragments which are not relevant to the example being discussed.  The readers can download or view all code from the above link.


<h2>Why do we need the SwingWorker?</h2>


Say we need to create an application that performs some task that take a considerable amount of time to complete.  Downloading a large file or executing a complex database query are good examples of such tasks.  Let us assume that these tasks are triggered by the users using a button.  With single threaded applications, the user clicks the button that starts the process and then has to wait for the task to finish before the user can do something else with the application.


<img src="http://www.javacreed.com/wp-content/uploads/2012/12/Swing-Worker-Example-1.png" alt="Single Thread" class="aligncenter size-full wp-image-4366" />


The application will become unresponsive as the only thread available is carrying the long task.  The user cannot even cancel the task halfway through neither, as the application is busy carrying the long task.  The application will become responsive only when the long task is finished.  Unfortunately many applications manifest such behaviour and users get frustrated as there is no way to cancel such task or interact with the application before this long task is complete.


Multithreading address this problem.  It enables the application to have the long tasks executed on a different thread.  The application handles small tasks, such as button clicks, by one thread and the long taking tasks by another thread.


<img src="http://www.javacreed.com/wp-content/uploads/2012/12/Swing-Worker-Example-2.png" alt="Two Threads" class="size-full wp-image-4367" />


This figure shows only two threads, but an application can have more than just two threads.  One can say that threads solved this problem and this is true.  But threads create new challenges.  The figure above shows two threads that work independently, that is, one thread does not communicate or exchange data with the other thread.  This may not always be the case and threads may need to share data.


Swing, like many other GUI frameworks, makes use of what is called thread confinement.  <strong>What is this?</strong>  All Swing objects are handled by one thread only, the event dispatcher thread.  <strong>We just agreed that multithreading yields more responsive applications, then why Swing uses one thread?</strong>  After several attempts (not only in Java but in many other languages too such as C++) in making GUI components multithreaded, it was decided to handle all GUI objects (such as buttons, lists, models and the like) by one thread.  All multithreaded GUI prototypes suffer from deadlocks due to reasons which are beyond the scope of this article.  For more information about this, please refer to this <a href="http://weblogs.java.net/blog/kgh/archive/2004/10/multithreaded_t.html" target="_blank">article</a>.


<strong>Therefore all Swing objects must be accessed only from the event dispatcher thread.</strong>


This leads back to the original problem.  We cannot share Swing objects with other threads outside the event dispatcher thread.  This is why we start a Swing application as shown below.


```java
package com.javacreed.examples.swing.worker.part1;

import javax.swing.JFrame;
import javax.swing.SwingUtilities;

public class Main {
  public static void main(final String[] args) {
      @Override
      public void run() {
        final JFrame frame = new JFrame();
        frame.setTitle("Test Frame");
        frame.setSize(600, 400);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setVisible(true);
      }
    });
  }
}
```


Here we are instructing Java to execute our <code>Runnable</code> (<a href="https://docs.oracle.com/javase/7/docs/api/java/lang/Runnable.html" target="_blank">Java Doc</a>) by the event dispatcher thread.  Therefore the <code>JFrame</code> (<a href="https://docs.oracle.com/javase/7/docs/api/javax/swing/JFrame.html" target="_blank">Java Doc</a>) exists only within this thread.  If all events are handled by one thread, then that thread will be block when we add long lasting tasks to the queue.  The <code>SwingWorker</code> offers a neat solution to this problem as we will see in this article.


<h2>SwingWorker</h2>


<code>SwingWorker</code> is an abstract class which hides the threading complexities from the developer.  It is an excellent candidate for applications that are required to execute tasks (such as retrieving information over the network/Internet or other slow sources) which may take some time to finish.  It is ideal to detach such tasks from the application and simply keep an <em>eye</em> on their progress and provide means for cancelling them.


The following example illustrates a simple empty worker that will return/evaluate to an integer when the given task is finished.  It will inform the application with what's happening using objects of type string, basically text messages.


```java
package com.javacreed.examples.swing.worker.part2;

import java.util.List;

import javax.swing.SwingWorker;

public class MyBlankWorker extends SwingWorker&lt;Integer, String&gt; {

  @Override
  protected Integer doInBackground() throws Exception {
    // Start
    publish("Start");
    setProgress(1);
    
    // More work was done
    publish("More work was done");
    setProgress(10);

    // Complete
    publish("Complete");
    setProgress(100);
    return 1;
  }
  
  @Override
  protected void process(List<String> chunks) {
    // Messages received from the doInBackground() (when invoking the publish() method)
  }
}
```


Let's understand the anatomy of the <code>SwingWorker</code> class. 


<ul>
<li>The <code>SwingWorker</code> class provides two placeholders (generics).  The first one represents the type of object returned when the worker has finished working.  The second one represents the type of information that the worker will use to inform (update) the application with its progress, and is shown in the following example.

```java
public class MyBlankWorker extends SwingWorker&lt;Integer, String&gt; {

  @Override
  protected Integer doInBackground() throws Exception {
    // Start
    publish("Start");
    setProgress(1);
    
    // More work was done
    publish("More work was done");
    setProgress(10);

    // Complete
    publish("Complete");
    setProgress(100);
    return 1;
  }
  
  @Override
  protected void process(List<String> chunks) {
    // Messages received from the doInBackground() (when invoking the publish() method)
  }
}
```
</li>

<li>
The <code>SwingWorker</code> class also provides means to update the progress by means of an <code>Integer</code> which has nothing to do with the two generics mentioned before.  This is managed through the <code>setProgress()</code> method which take an integer between 0 and 100 both inclusive.
</li>

<li>The method <code>doInBackground()</code> is where the complex and long task is executed.  This method is <strong>not</strong> invoked by the event dispatcher thread, but by another thread (referred to hereunder as the <em>worker thread</em>).  From this method we can update the progress using either the <code>publish()</code> method and/or the <code>setProgress()</code>.  Here something very important happens.  The invocations made by these two methods will add <em>small tasks</em> to the event dispatcher thread creating a one-way bridge between the thread that is doing the work and the event dispatcher thread.

The <code>doInBackground()</code> may invoke the <code>publish()</code> method and/or the <code>setProgress()</code> too often and may flood the event dispatcher thread queue.  In order to mitigate this problem, the <em>small tasks</em> may be suppressed or coalesced.  Therefore several requests may result in to one <em>small task</em> in the event dispatcher thread queue.
</li>

<li>
The method <code>publish()</code> is invoked several times from within the <code>doInBackground()</code> method and works together with the <code>process()</code>.  Further to what was described above, the <code>publish()</code> method is invoked from the worker thread, while the <code>process()</code> is invoked by the event dispatcher thread.  The following image illustrates this.

<img src="http://www.javacreed.com/wp-content/uploads/2012/12/Swing-Worker-Example-3.png" alt="Relation between the publish and process methods" width="916" height="599" class="size-full wp-image-4383" />

The long grey bar represents the <code>doInBackground()</code> method while the small red bars within it, represents the invocation of the <code>publish()</code> method.  As shown above, these requests may be coalesced into one request on the even dispatcher thread.  The two green bars represent the number of times the <code>process()</code> method is invoked.  In this example, the <code>publish()</code> method is invoked a total of 12 times, whereas the <code>process()</code> method is invoked only twice.  This also provides a performance boost as the event dispatcher thread is nor overloaded with too many small requests.

Furthermore, the <code>process()</code> method is invoked asynchronously on the event dispatch thread.  The above image shows this clearly and we have no control to when this is actually invoked.  It depends on the amount of work that the event dispatcher thread has.  

For example: 

```java
 publish("a");
 publish("b", "c");
 publish("d", "e", "f");
```

may result in: 

```java
 process("a", "b", "c", "d", "e", "f")
```

Since the <code>process()</code> method is invoked on the event dispatcher thread, we can access and modify the state of Swing components without having to worry about compromising the thread-safety.  We do not have to worry about thread-safety when using the Swing Worker as this has been dealt with by the same <code>SwingWorker</code> on our behalf.
</li>

<li>The <code>setProgress()</code> works very similar to what was described above, with one difference.  The application needs to register a listener in order to receive progress notifications.  This will be explored in more detail later on in this article.</li>
</ul>


Here we have described the functionality of each major block and the role these blocks play.  Now we will see a practical example of how we can utilise the <code>SwingWorker</code> within an application.


<h2>SwingWorker Example</h2>


Let say, for example, we need to find the number of occurrences of a given word (or phrase) with in some text documents located under a directory as shown in the following application.


<img src="http://www.javacreed.com/wp-content/uploads/2012/12/Swing-Worker-Example-5.png" alt="Search Word Application" width="888" height="827" class="size-full wp-image-4723" />


This application allows the user to provide the word to search and the path where to look for.  Once the user clicks the search button, the Swing worker starts searching in the background.  Thus it frees the event thread avoiding the application from freezing.  While waiting for the results, the user can cancel the task by pressing the cancel button that is shown instead of the search button.  


The swing working is a good candidate for such a problem.  The files listing and searching is quite a long task and can take a couple of seconds (if not minutes) to complete.  If this task is performed by the event dispatcher thread, then this thread will not be able to do anything else until this task is complete.  The user will not be able to cancel the task.


The following swing worker performs all the work we need.


```java
package com.javacreed.examples.swing.worker.part3;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

import javax.swing.JTextArea;
import javax.swing.SwingWorker;

import org.apache.commons.io.FileUtils;
import org.apache.commons.io.filefilter.SuffixFileFilter;
import org.apache.commons.io.filefilter.TrueFileFilter;
import org.apache.commons.lang.StringUtils;

/**
 * Searches the text files under the given directory and counts the number of instances a given word is found
 * in these file.
 * 
 * @author Albert Attard
 */
public class SearchForWordWorker extends SwingWorker&lt;Integer, String&gt; {

  private static void failIfInterrupted() throws InterruptedException {
    if (Thread.currentThread().isInterrupted()) {
      throw new InterruptedException("Interrupted while searching files");
    }
  }

  /** The word that is searched */
  private final String word;

  /**
   * The directory under which the search occurs. All text files found under the given directory are searched.
   */
  private final File directory;

  /** The text area where messages are written. */
  private final JTextArea messagesTextArea;

  /**
   * Creates an instance of the worker
   * 
   * @param word
   *          The word to search
   * @param directory
   *          the directory under which the search will occur. All text files found under the given directory
   *          are searched
   * @param messagesTextArea
   *          The text area where messages are written
   */
  public SearchForWordWorker(final String word, final File directory, final JTextArea messagesTextArea) {
    this.word = word;
    this.directory = directory;
    this.messagesTextArea = messagesTextArea;
  }

  @Override
  protected Integer doInBackground() throws Exception {
    // The number of instances the word is found
    int matches = 0;

    /*
     * List all text files under the given directory using the Apache IO library. This process cannot be
     * interrupted (stopped through cancellation). That is why we are checking right after the process whether
     * it was interrupted or not.
     */
    publish("Listing all text files under the directory: " + directory);
    final List<File> textFiles = new ArrayList<>(FileUtils.listFiles(directory, new SuffixFileFilter(".txt"),
        TrueFileFilter.TRUE));
    SearchForWordWorker.failIfInterrupted();
    publish("Found " + textFiles.size() + " text files under the directory: " + directory);

    for (int i = 0, size = textFiles.size(); i &lt; size; i++) {
      /*
       * In order to respond to the cancellations, we need to check whether this thread (the worker thread)
       * was interrupted or not. If the thread was interrupted, then we simply throw an InterruptedException
       * to indicate that the worker thread was cancelled.
       */
      SearchForWordWorker.failIfInterrupted();

      // Update the status and indicate which file is being searched.
      final File file = textFiles.get(i);
      publish("Searching file: " + file);

      /*
       * Read the file content into a string, and count the matches using the Apache common IO and Lang
       * libraries respectively.
       */
      final String text = FileUtils.readFileToString(file);
      matches += StringUtils.countMatches(text, word);

      // Update the progress
      setProgress((i + 1) * 100 / size);
    }

    // Return the number of matches found
    return matches;
  }

  @Override
  protected void process(final List&lt;String&gt; chunks) {
    // Updates the messages text area
    for (final String string : chunks) {
      messagesTextArea.append(string);
      messagesTextArea.append("\n");
    }
  }
}
```


Note that the swing worker takes a swing component (<code>JTextArea</code>) as an input to its constructor.  This swing component is only accessed from the <code>process()</code> method and never used from within the <code>doInBackbround()</code> method or other methods directly (by directly we mean from the same thread) invoked from it.  The following image highlights the places from where the swing component is accessed.


<img src="http://www.javacreed.com/wp-content/uploads/2012/12/Swing-Worker-Example-4.png" alt="Accessing Swing Components from the Swing Worker" width="1171" height="539" class="size-full wp-image-4736" />


This is very important as all swing components should be only accessed from the event dispatcher thread.


Now we saw how to create a swing worker.  In the next sections we will see how to start the worker and how to stop or better cancel the worker.

<h3>Starting the Swing Worker</h3>

The swing worker can be started by invoking the <code>execute()</code> method as shown in the following example.

```java
  private void search() {
    final String word = wordTextField.getText();
    final File directory = new File(directoryPathTextField.getText());
    messagesTextArea.setText("Searching for word '" + word + "' in text files under: " + directory.getAbsolutePath()
        + "\n");
    searchWorker = new SearchForWordWorker(word, directory, messagesTextArea);
    searchWorker.addPropertyChangeListener(new PropertyChangeListener() {
      @Override
      public void propertyChange(final PropertyChangeEvent event) {
        switch (event.getPropertyName()) {
        case "progress":
          searchProgressBar.setIndeterminate(false);
          searchProgressBar.setValue((Integer) event.getNewValue());
          break;
        case "state":
          switch ((StateValue) event.getNewValue()) {
          case DONE:
            searchProgressBar.setVisible(false);
            searchCancelAction.putValue(Action.NAME, "Search");
            try {
              final int count = searchWorker.get();
              JOptionPane.showMessageDialog(Application.this, "Found: " + count + " words", "Search Words",
                  JOptionPane.INFORMATION_MESSAGE);
            } catch (final CancellationException e) {
              JOptionPane.showMessageDialog(Application.this, "The search process was cancelled", "Search Words",
                  JOptionPane.WARNING_MESSAGE);
            } catch (final Exception e) {
              JOptionPane.showMessageDialog(Application.this, "The search process failed", "Search Words",
                  JOptionPane.ERROR_MESSAGE);
            }

            searchWorker = null;
            break;
          case STARTED:
          case PENDING:
            searchCancelAction.putValue(Action.NAME, "Cancel");
            searchProgressBar.setVisible(true);
            searchProgressBar.setIndeterminate(true);
            break;
          }
          break;
        }
      }
    });
    searchWorker.execute();
  }
```


This will create a new thread from where the <code>doInBackground()</code> method is invoked. 


<h3>Cancel the Worker</h3>


The swing worker can be stopped or better cancelled through the <code>cancel()</code> method. The swing worker provides a method called cancel which accepts a parameter of type <code>boolean</code>. This parameter determines whether or not the worker should be interrupted or not.


```java
  private void cancel() {
    searchWorker.cancel(true);
  }
```


This will cause the worker's <em>get</em> method to throw the exception <a href="http://java.sun.com/javase/7/docs/api/java/util/concurrent/CancellationException.html">CancellationException</a> to indicate that the worker was forced cancellation. </p>


<h2>Conclusion</h2>


Many GUI applications can be easily improved by taking advantage of the swing worker as we saw here.  Any tasks that may take long time to execute, should always be performed through a swing worker as these will hinder the application responsiveness.  Note that while testing, a developer may use a small search space which takes no time to execute, while in production the application will make use of a larger search space which will take longer to execute.  Thus may make the application unresponsive until the long task is finished.  The swing worker is not difficult to use and incorporate with existing code as shown in this article.
