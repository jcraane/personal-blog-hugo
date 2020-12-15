---
layout:     post

title:      "Melody composition with genetic algorithms"
subtitle:   "Creating musical pleasant melodies using genetic algorithms"
author: Jamie Craane
date:       2009-06-16
description: "Several techniques exist to create computer generated musical melodies. One of those techniques is genetic algorithms."
image: "/img/piano.jpg"
published: true
showtoc: false
tags:
- Machine learning

categories: [ Machine learning ]
URL: "/2009/06/16/ga_melody_composition/"
---
**Introduction**
Several techniques exist to create computer generated musical melodies. One of those techniques is genetic algorithms. Because the diversity of melodies over a specific range of notes is so large, genetic algorithms are a good candidate to help in composing melodies. This article describes how genetic algorithms can be used to compose musical melodies. this is explained following the steps needed to apply a genetic algorithm. These steps are:

1. Define the genetical representation of the problem
2. Determine the fitness function
3. Determine the parameters used for the run
4. Determine the termination criteria

The implementation of the genetic algorithm is done using the JGAP framework, a Java-based framework for implementing genetic algorithms. The generated melody itself is converted to MIDI which can then be played by the internal MIDI device or by a musical instrument attached to the computer

For more information about using genetic algorithms in Java please see my previous article which can be found here: [introduction to genetic algorithms](/2009/02/23/ga_introduction/). For more information about music theory which is applied in this article please refer to the excellent site http://www.musictheory.net.

**Define the genetical representation**
Before the genetic algorithm can do it's work, the genetical representation of the problem must be defined. To make things slightly less complicated the following constraints are introduced:

1. Only harmonic melodies are supported. This means that only one note is played at ones at any given time in the melody.
2. The generated melodies do not adhere to a specific measure. It is just a sequence of notes.

In a future version these constraints could be loosened.

A Melody can be seen as a composition of individual notes and rests. Those individual notes have properties which define how a certain note must be played. The following properties are used which are modified by the genetic algorithm:

- Pitch
  
    The pitch determines which notes on the grand staff is played. Possible values are C,D,E,F,G,A and B and all possible variants using sharps (#) and flats (b).
- Octave
  
    The octave determines in which octave a certain note is played. A piano has seven full octaves.
- Duration
  
    The length determines how long a certain note is played. Possible values are, whole notes, half notes, quarter notes, eight notes, sixteenth notes.

Notes have more properties than pitch and duration alone. Velocity, for example, indicates how hard a note is played. These properties are not used in this article. Besides notes, there are also rests in a melody. Rests can only have a duration.

A solution in the search space of melody generation is a melody consisting of individual notes. A solution is represented as a chromosome with a fixed number of genes. The genes in the chromosome represent the individual notes and rests, each with it's own characteristics. A note can be represented by using a composite gene with three integer gene's. The three integer gene's represent pitch, octave and duration. Integer gene's are chosen in favor of a custom gene implementation to make mutation easy and simple across the individual gene's. Each gene is described in detail here:

_Pitch_: The pitch can be represented as a number from 1 to 12. Since there are twelve semitones in an octave, each semitone can be represented by a number from 1 to 12. Every following number is equal to one semitone. The mapping of the notes is:

|Value|	Note|
|---|---|
|1	|C|
|2	|C# or Db|
|3	|D|
|4	|D# or Eb|
|5	|E|
|6	|F|
|7	|F# or Gb|
|8	|G|
|9	|G# or Ab|
|10	|A|
|11	|A# or Bb|
|12	|B|
|0	|rest|

Because all gene's are treated the same, a special case for the a rest in a melody is introduced. A rest has the value of 0 for the pitch. Since a rest is not associated with an octave, the octave property is ignored.

_Octave_: an octave can be represented by a number, the octave in which a certain note is played. The octave is indicated by a number from 1 to 7 where 1 represents the lowest octave on a piano keyboard and 7 the highest.

_Duration_: the duration can be represented by a number between 1 and 5. The mapping of the numbers are:

|Value|	Note|
|---|---|
|1|	Whole note|
|2|	Half note|
|3|	Quarter note|
|4|	Eighth note|
|5|	Sixteenth note|
                                         
The duration applies to both notes and rests. See the following code for the creation of the initial population of chromosomes:
```java
Configuration.reset();
Configuration gaConf = new DefaultConfiguration();
gaConf.resetProperty(Configuration.PROPERTY_FITEVAL_INST);
gaConf.setFitnessEvaluator(new DeltaFitnessEvaluator());

gaConf.setPreservFittestIndividual(true);
gaConf.setKeepPopulationSizeConstant(false);

gaConf.setPopulationSize(40);
CompositeGene gene = new CompositeGene(gaConf);
// Add the pitch gene
gene.addGene(new IntegerGene(gaConf, 0, 12), false);
// Add the octave gene
gene.addGene(new IntegerGene(gaConf, 1, 7), false);
// Add the length (from 3 - 5 is from quarter to sixteenth)
gene.addGene(new IntegerGene(gaConf, 1, 5), false);

// A size of 16 represent 16 notes
IChromosome sampleChromosome = new Chromosome(gaConf, gene, 16);
gaConf.setSampleChromosome(sampleChromosome);

gaConf.setFitnessFunction(melodyFitnessFunction);

return Genotype.randomInitialGenotype(gaConf);
```
##Determine the fitness function
The fitness function determines how good a specific melody is, relative to other melodies. The fitness function is the most complicated part of this problem since the fitness of a melody is subjective. Because of this nature, two approaches for fitness determination are presented.

**Computer generated fitness**

The computer generated fitness is purely based on certain algorithms to measure the fitness of a melody. Since there are a lot of different parameters, the fitness function combines the fitness value of several different strategies which can be easily added to the fitness function. Some of these parameters, but not limited to, are:

- 80% of the notes in a melody may not have more than than 7 semitones difference.
- A melody may not span more than 2 octaves
- A melody must be in C-major (or minor scale)
- Only 10% of a melody may consist of rests
- Two consecutive notes may not lie more than 5 semitones from each other
- etc.

In the proof-of-concept implementation the following rules regarding the fitness of a melody are implemented. These rules are implemented as separate classes which all implement the MelodyFitnessStrategy interface. This makes it easy and straightforward to add more rules which measure the fitness of a melody.

- ScaleStrategy
  
    Calculates if a given melody adheres to a specific scale, for example C major. The scale can be set as a parameter on this class.
- IntervalStrategy
  
    Calculates if a given melody has one or more major and/or perfect intervals. The number of major and perfect intervals can be set as parameters on this class.
- GlobalPitchDistributionStrategy
  
    Calculates if the lowest and highest pitch of a given melody fall within the margins specified by this class. The margin is indicated as the number of semitones and a percentage about how much of the notes must fall between the given semitones. These values can be set as parameters on this class.
- RepeatingNotesStrategy
  
    Calculates if a given melody has repeating notes or rests. The maximum number of repeating notes and/or rests can be set as parameters on this class.
- PropertionRestAndNotesStrategy
  
    Calculates the proportion between the notes and rests in a given melody. The propertion of notes/rests can be set as parameter on this class.
- ParallelIntervalStrategy
  
    Calculates the number of parallel intervals in this melody. Some parallel intervals are supposed to sound good, like thirds and sixths. The number of good sounding parallel intervals can be set as a parameter on this class.

A builder class exists which helps in the creation of a valid fitness function.

Please note that all rules calculate the deviation between the generated melody and the specified rules. A lower fitness value means less deviation which means a better melody. In the future it is planned to add more strategies to calculate the fitness of a given melody. For example:

- ContourStrategy. Strategy which calculates if a given melody has a specific contour in the pitch of the individual notes.

Below is the implementation of the ScaleStrategy:
```java
public final class ScaleStrategy extends AbstractMelodyFitnessStrategy {
  private static final int ERROR_COUNT_WHEN_NOTE_NOT_ON_SCALE = 1;

  @Override
  // The more difference between current note and note in scale the higher the error count
  public double calculateErrors(IChromosome melody) {
     double errors = 0.0D;
     for (Gene gene : melody.getGenes()) {
        Note note = GeneNoteFactory.fromGene((CompositeGene) gene);
        if (Pitch.REST != note.getPitch() && !super.scale.contains(note.getPitch())) {
           errors += ERROR_COUNT_WHEN_NOTE_NOT_ON_SCALE;
        }
     }

     // Adhering to a given scale is quite important so square the result
     return (errors * errors) * 10;
  }

  public String toString() {
     return "[ScaleStrategy[scale: " + this.scale + "]]";
  }
}
```
**A different approach**

The above paragraph describes how the fitness of a melody can be computed based on the knowledge of music theory. Instead of using music theory to compute the fitness of a melody, a different approach can be used. This approach looks at the melody as audio data, an array of bytes, instead as a sequence of notes. From this audio data, different information can be extracted. For example, by using a Fast Fourier Transform (FFT), the audio data can be viewed in the frequency domain. By analyzing existing melodies using FFT, an algorithm can be constructed which measures the fitness of generated melodies based on this knowledge. A future version of the application may use the technique described here.

**Human intervention based fitness**

Since melodies are very subjective it is hard to come up with a computer based mechanism to measure how good a specific melody is. Another approach which can be used is a human based intervention fitness function. In this approach the user is represented with a fixed set of melodies generated by the genetic algorithm. The user selects two or three melodies which participate in the next evolution. Although this approach is not implemented in the current version, expect a future release to use this approach.
##Determine the run parameters
When using a computer based fitness function, several parameters which affect the generated melody can be provided by the user. See the section about the computer generated fitness function which parameters can be supplied. Based on some sample runs, the number of evolutions to execute is 250. 250 evolutions seems like a good trade off between quality of the generated melody and computation time. The quality is measured in terms of the fitness value of the melody. With 250 evolutions, the fitness value of a 24-note melody almost always approaches zero.
##Determine the termination criteria
The run ends when 250 evolutions are executed.

**Some sample runs**

The program can be executed in two different ways:

1. By executing the MelodyGeneratorMain class file from the command line. Modify and recompile this class to alter the parameters of the run, for example the fitness function.
2. Via the Swing UI. This is a very simple Swing UI build with Groovy's SwingBuilder. All of the parameters of the fitness function can be modified with this UI. Please note that this is a very simple UI implementation and not an example of how to write production quality Swing code.

To generate a melody from the UI, just click the generate button. When finished, the application plays back the melody and gives the option to replay, save or generate a new melody. Just experiment with the different settings and listen to the various generated melodies. In the UI you can specify the path to write the MIDI files to. Make sure this path exists on disk.

**Conclusion**

This article explains how genetic algorithms can be used to compose melodies. Genetic algorithms seem like a viable alternative for melody generation since they are very well suited to search for specific solutions in very large search spaces. In this case the search space are all possible combination of notes. The difficulty in generating melodies with genetic algorithms is the specification of the fitness function since this is very subjective.
Although the melodies generated by this program are nowhere near hit melodies, they can be used as inspiration when composing melodies. The performance of the algorithm can be improved by writing additional (and more complex) strategies to measure the fitness of a melody.

Let me know what ideas you have to measure the fitness of the generated melodies to improve the quality of the melodies.

The full source code of this application can be found on Google code, see the resources.

**Resources**
- https://github.com/jcraane/melodycomposition_genetic
- www.musictheory.net
