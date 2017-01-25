# bluenet-basic-localization-ios
Basic version of the bluenet localization for iOS.

### Installation

This module is made to be used with the BluenetLibIOS which is found here:
https://github.com/crownstone/bluenet-lib-ios

To use, you can use the zip file in the releases found here:
https://github.com/crownstone/bluenet-basic-localization-ios/releases

Download the zip file in the latest release and extract it. From the archive /Carthage/Build/iOS get the following files:
- BluenetBasicLocalization.framework
- BluenetBasicLocalization.framework.dSYM
Place these in the ./Carthage/Build/iOS folder of the project you added BluenetLibIOS too and include the .framework file like a Carthage framework.

### Creating your own classifier

To implement your own classifier, you have to make sure your classifier adheres to the protocols hosted here:
https://github.com/crownstone/bluenet-ios-shared/

The protocols are used to allow Bluenet to communicate with the classifier.

A minimal classifier will look like this:

```swift

open class CrownstoneBasicClassifier: LocalizationClassifier {
    public init() {}
    
    public func loadTrainingData(_ dataId: String, collectionId: String, trainingData: <someDataType>) {
        /* 
        A method to load training data into the classifier. 
        This can be called multiple times to load different training sets into the classifier.
        Each set is a different possibility.
        The id provided will be returned as string by the classify method
        */        
    }
    
    public func classify(_ inputVector: [iBeaconPacketProtocol], collectionId: String) -> String? {
        return <result>
    }
    
}
```
