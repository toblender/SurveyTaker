# Uncomment the next line to define a global platform for your project
platform :ios, '10.0'

def testing_pods
  # pod for test targets
end

target 'SurveyTaker' do
  use_frameworks!

  pod 'Fabric'
  pod 'Crashlytics'
  pod 'SwiftLint'

  target 'SurveyTakerTests' do
    inherit! :search_paths
    testing_pods
  end

  target 'SurveyTakerUITests' do
    inherit! :search_paths
    testing_pods
  end

  def swift_4_pods
    []
  end
end

def isSwift4?(pod_name)
  swift_4_pods.include? pod_name
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    if isSwift4? target.name
      target.build_configurations.each do |config|
        config.build_settings['SWIFT_VERSION'] = '4.0'
      end
    end
  end
end
